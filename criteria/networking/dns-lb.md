# DNS and Load Balancing Reference Checklist

Canonical reference for evaluating DNS server configuration, DNSSEC deployment, zone transfer
security, and load balancer design. Complements the inlined criteria in `review-networking.md`
with CCIE-equivalent depth covering DNS security, load balancer health checks, SSL/TLS
termination, persistence methods, and global server load balancing (GSLB).

Covers: DNS server hardening, DNSSEC, split-horizon DNS, DNS over TLS/HTTPS, L4 and L7
load balancing, health monitoring, SSL/TLS offload, connection management, and GSLB.

---

## 1. DNS Server Configuration

### 1.1 General Hardening

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | DNS server software is current and patched (BIND, Unbound, PowerDNS, Windows DNS, Knot) | **HIGH** |
| 1.1.2 | DNS server runs as a non-root user with minimal privileges | **HIGH** |
| 1.1.3 | DNS server runs in a chroot or sandboxed environment where supported | **MEDIUM** |
| 1.1.4 | Recursion is disabled on authoritative-only servers | **CRITICAL** |
| 1.1.5 | Open recursion is not available to the public internet (recursive resolvers restricted to internal clients) | **CRITICAL** |
| 1.1.6 | DNS query logging is enabled for security monitoring | **MEDIUM** |
| 1.1.7 | DNS cache size is appropriately configured to prevent memory exhaustion | **MEDIUM** |
| 1.1.8 | Version string is hidden to prevent information disclosure | **LOW** |
| 1.1.9 | CHAOS class queries are disabled or restricted | **LOW** |
| 1.1.10 | DNS server binds to specific interfaces, not all interfaces | **MEDIUM** |

### What to Look For

```regex
# BIND recursion enabled on authoritative server
(?i)recursion\s+yes\s*;
# On authoritative-only servers, this should be "recursion no;"

# Open recursion (no allow-recursion ACL)
(?i)recursion\s+yes\s*;
# Verify: allow-recursion { <trusted-networks>; }; exists

# Version disclosure
(?i)version\s+"[^"]*"\s*;
# Should be: version "none"; or version "not disclosed";

# BIND running as root
(?i)named\s+.*-u\s+root
# Should run as: named -u named (or bind user)

# Missing query logging
(?i)logging\s*\{
# Verify: queries channel and category exist

# Listen on all interfaces
(?i)listen-on\s*\{\s*any\s*;\s*\}
# Should specify: listen-on { 10.0.0.1; };
```

### Vendor Examples

**BIND named.conf hardening**:
```
options {
    directory "/var/named";
    version "not disclosed";
    hostname "not disclosed";

    // Restrict recursion to internal clients
    recursion yes;
    allow-recursion { 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16; };

    // Bind to specific interfaces
    listen-on { 10.0.0.1; 127.0.0.1; };
    listen-on-v6 { ::1; };

    // Rate limiting
    rate-limit {
        responses-per-second 10;
        window 5;
    };

    // Prevent cache poisoning
    dnssec-validation auto;

    // Disable CHAOS class
    allow-query-cache-on { none; };

    // Minimize responses
    minimal-responses yes;
};

logging {
    channel queries_log {
        file "/var/log/named/queries.log" versions 5 size 50m;
        severity info;
        print-time yes;
        print-severity yes;
    };
    category queries { queries_log; };
};
```

**Unbound configuration hardening**:
```
server:
    interface: 10.0.0.1
    access-control: 10.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow
    access-control: 0.0.0.0/0 refuse

    hide-identity: yes
    hide-version: yes

    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-referral-path: yes
    harden-algo-downgrade: yes

    use-caps-for-id: yes
    val-clean-additional: yes

    unwanted-reply-threshold: 10000

    prefetch: yes
    prefetch-key: yes

    rrset-roundrobin: yes
    minimal-responses: yes

    auto-trust-anchor-file: "/var/lib/unbound/root.key"

    log-queries: yes
    verbosity: 1
```

---

## 2. Zone Transfer Security

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 2.1 | Zone transfers (AXFR/IXFR) are restricted to authorized secondary DNS servers by IP | **CRITICAL** |
| 2.2 | TSIG (Transaction Signature, RFC 8945) is used for zone transfer authentication | **HIGH** |
| 2.3 | Zone transfer ACLs use specific IP addresses, not network ranges | **HIGH** |
| 2.4 | Dynamic DNS updates are restricted to authorized sources with TSIG | **HIGH** |
| 2.5 | Zone transfer notifications (NOTIFY) are limited to known secondaries | **MEDIUM** |
| 2.6 | AXFR availability is periodically tested to verify it is properly restricted | **LOW** |

### What to Look For

```regex
# Unrestricted zone transfers
(?i)allow-transfer\s*\{\s*any\s*;\s*\}
# Should be: allow-transfer { <secondary-ip>; key <tsig-key>; };

# Missing transfer restriction (default allows all in some configurations)
(?i)zone\s+"[^"]+"\s*\{
# Verify: allow-transfer directive exists

# Dynamic updates unrestricted
(?i)allow-update\s*\{\s*any\s*;\s*\}
# Should use TSIG key

# Missing TSIG
(?i)allow-transfer\s*\{\s*\d+\.\d+\.\d+\.\d+
# Verify: key directive also present for authentication
```

### Vendor Examples

**BIND zone with TSIG and transfer restriction**:
```
key "transfer-key" {
    algorithm hmac-sha256;
    secret "base64-encoded-key==";
};

zone "example.com" {
    type master;
    file "/var/named/example.com.zone";
    allow-transfer { key "transfer-key"; 10.0.1.2; };
    also-notify { 10.0.1.2; };
    allow-update { key "ddns-key"; };
};
```

**PowerDNS zone transfer restriction**:
```
# pdns.conf
allow-axfr-ips=10.0.1.2/32,10.0.1.3/32
disable-axfr=no
# TSIG configured per zone in database or config
```

**Windows DNS zone transfer restriction**:
```powershell
# Restrict zone transfers to named servers only
Set-DnsServerPrimaryZone -Name "example.com" `
    -SecureSecondaries TransferToSecureServers `
    -SecondaryServers @("10.0.1.2","10.0.1.3")
```

---

## 3. DNSSEC

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | DNSSEC validation is enabled on recursive resolvers | **HIGH** |
| 3.2 | DNSSEC signing is deployed on authoritative zones for external domains | **MEDIUM** |
| 3.3 | KSK (Key Signing Key) and ZSK (Zone Signing Key) are properly separated | **MEDIUM** |
| 3.4 | DNSSEC key rotation schedule is documented and automated | **MEDIUM** |
| 3.5 | DS records are published in the parent zone for chain of trust | **HIGH** |
| 3.6 | NSEC3 is used instead of NSEC to prevent zone enumeration | **MEDIUM** |
| 3.7 | DNSSEC algorithm is current (RSA-SHA256 minimum, ECDSA P-256 or Ed25519 preferred) | **MEDIUM** |
| 3.8 | DNSSEC key material is stored securely (HSM for high-value zones) | **HIGH** |
| 3.9 | DNSSEC monitoring alerts on signature expiration | **HIGH** |
| 3.10 | Trust anchors are automatically maintained (RFC 5011) | **MEDIUM** |

### What to Look For

```regex
# DNSSEC validation enabled (positive indicator)
(?i)dnssec-validation\s+(auto|yes)\s*;

# DNSSEC validation disabled
(?i)dnssec-validation\s+no\s*;

# DNSSEC signing indicators
(?i)(dnssec-signzone|auto-dnssec\s+maintain|dnssec-policy)

# Weak DNSSEC algorithm
(?i)(RSAMD5|DSA|RSASHA1)\b
# Prefer: RSASHA256, RSASHA512, ECDSAP256SHA256, ED25519

# Missing NSEC3
(?i)NSEC\s+
# Check: should be NSEC3 with opt-out if applicable
```

### Vendor Examples

**BIND DNSSEC signing (inline signing)**:
```
zone "example.com" {
    type master;
    file "/var/named/example.com.zone";
    key-directory "/var/named/keys";
    auto-dnssec maintain;
    inline-signing yes;
    allow-transfer { key "transfer-key"; 10.0.1.2; };
};

// Or with BIND 9.16+ dnssec-policy:
zone "example.com" {
    type master;
    file "/var/named/example.com.zone";
    dnssec-policy "default";
    inline-signing yes;
};
```

**Unbound DNSSEC validation**:
```
server:
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
    val-clean-additional: yes
    harden-dnssec-stripped: yes
```

---

## 4. Split-Horizon DNS

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | Internal and external DNS views are properly separated | **HIGH** |
| 4.2 | Internal hostnames, IP addresses, and infrastructure details are not exposed in external DNS | **HIGH** |
| 4.3 | View matching uses source IP ACLs to correctly classify internal vs. external clients | **HIGH** |
| 4.4 | Internal-only records (management IPs, database servers, internal APIs) are not resolvable externally | **HIGH** |
| 4.5 | Split-horizon configuration is tested from both internal and external perspectives | **MEDIUM** |
| 4.6 | PTR records in internal ranges do not leak information externally | **MEDIUM** |

### What to Look For

```regex
# BIND view configuration (positive indicator)
(?i)view\s+"(internal|external|public|private)"\s*\{

# Missing view match-clients ACL
(?i)view\s+"[^"]+"\s*\{(?:(?!match-clients)[^}])*\}
# Verify: match-clients directive with appropriate ACL

# Internal records in external zone files
(?i)\b(10\.\d+\.\d+\.\d+|172\.(1[6-9]|2\d|3[01])\.\d+\.\d+|192\.168\.\d+\.\d+)\b
# In external zone files, RFC 1918 addresses should not appear
```

### Vendor Examples

**BIND split-horizon**:
```
acl "internal-clients" {
    10.0.0.0/8;
    172.16.0.0/12;
    192.168.0.0/16;
    localhost;
};

view "internal" {
    match-clients { internal-clients; };
    recursion yes;

    zone "example.com" {
        type master;
        file "/var/named/internal/example.com.zone";
    };
};

view "external" {
    match-clients { any; };
    recursion no;

    zone "example.com" {
        type master;
        file "/var/named/external/example.com.zone";
    };
};
```

---

## 5. DNS over TLS (DoT) and DNS over HTTPS (DoH)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 5.1 | DNS over TLS (RFC 7858, port 853) or DNS over HTTPS (RFC 8484, port 443) is available for internal clients | **MEDIUM** |
| 5.2 | DoT/DoH server certificates are valid, from a trusted CA, and auto-renewed | **HIGH** |
| 5.3 | Plaintext DNS (port 53) is restricted or monitored when DoT/DoH is deployed | **MEDIUM** |
| 5.4 | DoT/DoH resolvers log queries for security monitoring (encrypted DNS does not mean unmonitored) | **MEDIUM** |
| 5.5 | Client configuration is managed centrally (not relying on individual device settings) | **LOW** |
| 5.6 | Fallback to plaintext DNS is controlled and documented | **LOW** |

### Vendor Examples

**Unbound DoT configuration**:
```
server:
    interface: 10.0.0.1@853
    tls-service-key: "/etc/unbound/server.key"
    tls-service-pem: "/etc/unbound/server.pem"
    tls-port: 853
```

**BIND DoT/DoH (9.18+)**:
```
options {
    listen-on tls tls-config { 10.0.0.1; };
    listen-on-v6 tls tls-config { ::1; };
    http-port 443;
    http-listener-clients 100;
};

tls tls-config {
    key-file "/etc/bind/tls/server.key";
    cert-file "/etc/bind/tls/server.pem";
};
```

---

## 6. Load Balancer Health Checks

### 6.1 Health Check Design

| # | Checkpoint | Severity |
|---|---|---|
| 6.1.1 | Health checks test application functionality, not just TCP connectivity | **HIGH** |
| 6.1.2 | Health check endpoints return status based on actual service health (database, dependencies) | **HIGH** |
| 6.1.3 | Health check intervals are appropriate (5-30 seconds for active checks) | **MEDIUM** |
| 6.1.4 | Health check timeout is shorter than the interval (prevents overlapping probes) | **MEDIUM** |
| 6.1.5 | Failure threshold (consecutive failures before marking down) is set to avoid flapping (typically 3) | **MEDIUM** |
| 6.1.6 | Recovery threshold (consecutive successes before marking up) prevents premature reinsertion | **MEDIUM** |
| 6.1.7 | Health check source IP is allowed through backend firewalls and security groups | **HIGH** |
| 6.1.8 | Health check endpoint is separate from production endpoints (dedicated `/health` or `/healthz`) | **MEDIUM** |
| 6.1.9 | Health checks do not cause side effects or generate excessive load on backends | **MEDIUM** |
| 6.1.10 | Health check responses include degraded states (not just up/down) | **LOW** |

### What to Look For

```regex
# TCP-only health check (insufficient)
(?i)(monitor|health.?check|probe)\s+.*\b(tcp|port)\b
# Should include: HTTP GET, application-level check

# Missing health check entirely
(?i)(pool|server.?group|backend)\s+\S+
# Verify: associated monitor/health-check defined

# Health check interval and timeout
(?i)(interval|frequency)\s*[:=]\s*(\d+)
(?i)timeout\s*[:=]\s*(\d+)
# Verify: timeout < interval

# Health check endpoint patterns
(?i)(health|healthz|status|ready|readiness|liveness|heartbeat)
```

### Vendor Examples

**F5 BIG-IP health monitor**:
```
ltm monitor http MONITOR-APP-HEALTH {
    defaults-from http
    interval 10
    timeout 31
    recv "\"status\":\"healthy\""
    send "GET /health HTTP/1.1\r\nHost: app.example.com\r\nConnection: close\r\n\r\n"
    up-interval 0
}

ltm pool POOL-APP {
    members {
        10.1.1.10:443 {}
        10.1.1.11:443 {}
    }
    monitor MONITOR-APP-HEALTH
    min-active-members 1
}
```

**HAProxy health check**:
```
backend app_servers
    option httpchk GET /health HTTP/1.1\r\nHost:\ app.example.com
    http-check expect status 200

    server app1 10.1.1.10:8080 check inter 10s fall 3 rise 2
    server app2 10.1.1.11:8080 check inter 10s fall 3 rise 2
```

**NGINX health check (Plus/commercial)**:
```
upstream app_servers {
    zone app_servers 64k;
    server 10.1.1.10:8080;
    server 10.1.1.11:8080;
}

server {
    location / {
        proxy_pass http://app_servers;
        health_check interval=10 fails=3 passes=2 uri=/health;
    }
}
```

**AWS ALB health check (Terraform)**:
```hcl
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 15
    timeout             = 5
    matcher             = "200"
  }
}
```

---

## 7. Persistence (Session Affinity)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | Persistence method is appropriate for the application (cookie, source-IP, SSL session ID) | **HIGH** |
| 7.2 | Cookie-based persistence is preferred over source-IP for web applications | **MEDIUM** |
| 7.3 | Persistence timeout is set to match application session timeout | **MEDIUM** |
| 7.4 | Persistence does not defeat load distribution (monitor distribution across backends) | **MEDIUM** |
| 7.5 | Source-IP persistence is not used behind NAT or proxy (all clients appear as same IP) | **HIGH** |
| 7.6 | Persistence cookies are flagged Secure and HttpOnly when inserted by the LB | **HIGH** |
| 7.7 | Persistence table size is monitored to prevent exhaustion | **MEDIUM** |
| 7.8 | Application is designed to be stateless where possible, reducing persistence dependency | **MEDIUM** |

### What to Look For

```regex
# Source-IP persistence behind NAT (problematic)
(?i)(source.?addr|src.?ip|source.?ip)\s*(persistence|affinity|sticky)
# Verify: not used behind NAT/proxy where many clients share an IP

# Missing persistence cookie flags
(?i)(cookie|persistence)\s+
# Verify: Secure and HttpOnly flags set

# No persistence configured (may be intentional for stateless apps)
(?i)(pool|backend|upstream)\s+\S+
# Check: does the application require session affinity?
```

### Vendor Examples

**F5 BIG-IP cookie persistence**:
```
ltm persistence cookie PERSIST-APP-COOKIE {
    cookie-name "BIG-IP-PERSIST"
    expiration 0
    method insert
    cookie-encryption required
    httponly enabled
    secure enabled
}

ltm virtual VS-APP {
    destination 192.168.1.100:443
    pool POOL-APP
    persist { PERSIST-APP-COOKIE { default yes } }
}
```

**HAProxy cookie persistence**:
```
backend app_servers
    cookie SERVERID insert indirect nocache httponly secure
    server app1 10.1.1.10:8080 cookie s1 check
    server app2 10.1.1.11:8080 cookie s2 check
```

**NGINX sticky cookie (Plus)**:
```
upstream app_servers {
    server 10.1.1.10:8080;
    server 10.1.1.11:8080;
    sticky cookie srv_id expires=1h domain=.example.com httponly secure path=/;
}
```

---

## 8. SSL/TLS Termination

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 8.1 | TLS 1.2 is the minimum protocol version; TLS 1.3 is preferred | **HIGH** |
| 8.2 | SSLv3, TLS 1.0, and TLS 1.1 are disabled | **HIGH** |
| 8.3 | Cipher suites exclude weak algorithms (RC4, DES, 3DES, MD5, export ciphers, NULL) | **HIGH** |
| 8.4 | Forward secrecy (ECDHE, DHE) is enforced in cipher suite ordering | **HIGH** |
| 8.5 | Certificates are from a trusted CA (not self-signed in production) | **HIGH** |
| 8.6 | Certificate chain is complete (includes intermediate certificates) | **MEDIUM** |
| 8.7 | Certificate expiration is monitored with alerts at 30, 14, and 7 days | **HIGH** |
| 8.8 | OCSP stapling is enabled for certificate revocation checking | **MEDIUM** |
| 8.9 | HSTS header is set with `max-age >= 31536000` and `includeSubDomains` | **HIGH** |
| 8.10 | Private keys are stored securely (HSM, key vault, encrypted at rest) | **CRITICAL** |
| 8.11 | SSL/TLS offload does not break end-to-end encryption for sensitive data paths (re-encrypt to backend) | **HIGH** |
| 8.12 | X-Forwarded-For and X-Forwarded-Proto headers are set for backend visibility | **MEDIUM** |
| 8.13 | Client certificate authentication (mTLS) is used for service-to-service traffic | **MEDIUM** |
| 8.14 | HTTP to HTTPS redirect is configured (301 redirect) | **HIGH** |

### What to Look For

```regex
# Weak TLS versions
(?i)(SSLv2|SSLv3|TLSv1\.0|TLSv1\.1|ssl3|tls1\.0|tls1\.1)
(?i)(ssl_protocols|ssl-protocol|protocols)\s+.*\b(SSLv|TLSv1\b|TLSv1\.0|TLSv1\.1)

# Weak cipher suites
(?i)(RC4|DES|3DES|MD5|NULL|EXPORT|aNULL|eNULL|DES-CBC3)
(?i)ssl_ciphers\s+.*\b(RC4|DES|3DES|aNULL|eNULL|EXPORT)\b

# Self-signed certificate in production
(?i)(self.?signed|selfsigned)

# Missing HSTS
(?i)(Strict-Transport-Security|hsts)
# Verify: max-age is >= 31536000

# Private key in plaintext
(?i)(BEGIN\s+RSA\s+PRIVATE\s+KEY|BEGIN\s+EC\s+PRIVATE\s+KEY|BEGIN\s+PRIVATE\s+KEY)

# Missing OCSP stapling
(?i)(ocsp.?stapl|ssl_stapling)
```

### Vendor Examples

**F5 BIG-IP TLS profile**:
```
ltm profile client-ssl PROFILE-CLIENTSSL-SECURE {
    cert /Common/wildcard.example.com.crt
    key /Common/wildcard.example.com.key
    chain /Common/intermediate-ca.crt
    ciphers !SSLv2:!SSLv3:!TLSv1:!TLSv1_1:ECDHE+AES-GCM:ECDHE+AES:!RC4:!3DES:!aNULL:!eNULL:!EXPORT
    options { dont-insert-empty-fragments no-ssl-v2 no-ssl-v3 no-tls-v1 no-tls-v1.1 }
}
```

**HAProxy TLS configuration**:
```
global
    ssl-default-bind-ciphersuites TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    tune.ssl.default-dh-param 2048

frontend https
    bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    http-request redirect scheme https unless { ssl_fc }
```

**NGINX TLS configuration**:
```
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_stapling on;
ssl_stapling_verify on;

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

---

## 9. Connection Limits and Rate Limiting

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 9.1 | Maximum concurrent connections per virtual server are configured | **HIGH** |
| 9.2 | Per-source connection limits prevent a single client from exhausting capacity | **HIGH** |
| 9.3 | Connection rate limiting prevents SYN flood and application-layer DoS | **HIGH** |
| 9.4 | Idle connection timeouts are set to reclaim resources | **MEDIUM** |
| 9.5 | HTTP keep-alive timeout is reasonable (not indefinite) | **MEDIUM** |
| 9.6 | Slow POST/Slowloris protections are enabled on HTTP load balancers | **HIGH** |
| 9.7 | Connection queue depth is configured with a reasonable limit | **MEDIUM** |
| 9.8 | Connection table monitoring and alerting is configured | **MEDIUM** |
| 9.9 | SSL/TLS renegotiation is disabled or rate-limited (CPU-intensive) | **MEDIUM** |

### What to Look For

```regex
# Missing connection limits
(?i)(virtual.?server|frontend|server)\s+
# Verify: connection-limit, maxconn, or equivalent exists

# No idle timeout
(?i)(idle.?timeout|timeout\s+client|timeout\s+server)
# Verify: reasonable values (not 0 or extremely high)

# Slowloris protection
(?i)(slow.?post|slowloris|reqtimeout|client.?header.?timeout|http-request-timeout)

# SSL renegotiation
(?i)(ssl.?renegotiat|renegotiate)
```

### Vendor Examples

**HAProxy connection management**:
```
defaults
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-request 10s
    timeout http-keep-alive 5s
    timeout queue 30s
    maxconn 50000

frontend https
    bind *:443 ssl crt /etc/haproxy/certs/
    maxconn 50000

    # Rate limiting
    stick-table type ip size 100k expire 30s store conn_cur,conn_rate(10s),http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_conn_rate(0) gt 100 }
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 500 }
```

**F5 BIG-IP connection limits**:
```
ltm virtual VS-APP {
    destination 192.168.1.100:443
    connection-limit 100000
    rate-limit 10000
    rate-limit-mode destination
}

ltm pool POOL-APP {
    members {
        10.1.1.10:8080 {
            connection-limit 5000
        }
        10.1.1.11:8080 {
            connection-limit 5000
        }
    }
}
```

**NGINX connection and rate limiting**:
```
# Connection limit zone
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;

# Request rate limit zone
limit_req_zone $binary_remote_addr zone=apilimit:10m rate=10r/s;

server {
    # Per-IP connection limit
    limit_conn perip 100;
    # Per-server connection limit
    limit_conn perserver 10000;

    location /api/ {
        limit_req zone=apilimit burst=20 nodelay;
        proxy_pass http://app_servers;
    }
}
```

---

## 10. L4 vs. L7 Load Balancing

### 10.1 Layer 4 Load Balancing

| # | Checkpoint | Severity |
|---|---|---|
| 10.1.1 | L4 load balancing is used for non-HTTP protocols (database, custom TCP/UDP) | **MEDIUM** |
| 10.1.2 | L4 load balancing algorithm is appropriate (round-robin, least-connections, weighted) | **MEDIUM** |
| 10.1.3 | DSR (Direct Server Return) is configured where performance requires it and topology supports it | **LOW** |
| 10.1.4 | Source NAT (SNAT) is configured when backends need to see the load balancer as the gateway | **MEDIUM** |
| 10.1.5 | SNAT pool is sized to avoid port exhaustion (> 64000 connections per SNAT IP) | **HIGH** |

### 10.2 Layer 7 Load Balancing

| # | Checkpoint | Severity |
|---|---|---|
| 10.2.1 | Content-based routing uses reliable headers (Host, path, not easily spoofed values) | **MEDIUM** |
| 10.2.2 | HTTP/2 is enabled on the frontend for improved client performance | **MEDIUM** |
| 10.2.3 | Backend connection reuse (multiplexing) is enabled to reduce TCP overhead | **MEDIUM** |
| 10.2.4 | Request and response buffering is configured appropriately | **LOW** |
| 10.2.5 | Error pages are customized (no backend stack traces exposed) | **MEDIUM** |
| 10.2.6 | WAF integration is in place for public-facing virtual servers | **HIGH** |
| 10.2.7 | HTTP request smuggling protections are enabled (normalize request line, reject ambiguous requests) | **HIGH** |
| 10.2.8 | X-Forwarded-For header is properly managed (append, not blindly trust) | **MEDIUM** |

### What to Look For

```regex
# SNAT pool size check
(?i)(snat|source.?nat|src.?nat)\s+
# Verify: multiple IPs in SNAT pool for high-connection environments

# HTTP/2 support
(?i)(h2|http2|http/2|alpn)

# Request smuggling protections
(?i)(http-request-smuggling|request.?smuggl|normalize|reject.?malformed)

# Error page configuration
(?i)(errorfile|error.?page|custom.?error|maintenance.?page)
```

---

## 11. Global Server Load Balancing (GSLB)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 11.1 | GSLB uses authoritative DNS responses with appropriate TTLs (30-300 seconds) | **MEDIUM** |
| 11.2 | GSLB health monitors verify application availability at each site, not just IP reachability | **HIGH** |
| 11.3 | GSLB failover logic is documented and tested (active-active, active-standby, priority-based) | **HIGH** |
| 11.4 | GSLB topology records or geo-IP databases are current | **MEDIUM** |
| 11.5 | GSLB persistence (site affinity) is configured to prevent session disruption during failover | **MEDIUM** |
| 11.6 | DNS TTL for GSLB records allows timely failover (not > 300 seconds for DR scenarios) | **HIGH** |
| 11.7 | GSLB inter-site communication is encrypted and authenticated | **HIGH** |
| 11.8 | Manual failover capability exists for planned maintenance | **MEDIUM** |
| 11.9 | GSLB monitors multiple data points (round-trip time, site capacity, health) for intelligent decisions | **MEDIUM** |
| 11.10 | Fallback (last-resort) pool is configured for when all sites fail health checks | **HIGH** |

### What to Look For

```regex
# GSLB DNS TTL too high
(?i)(ttl|time.?to.?live)\s*[:=]\s*(\d{4,})
# TTLs above 300 seconds may delay failover

# Missing GSLB health monitors
(?i)(gslb|gtm|global|wide.?ip)\s+
# Verify: health monitor attached to each pool member

# GSLB inter-site encryption
(?i)(iquery|sync.?group|gslb.?sync|data.?center)
# Verify: encrypted communication between GSLB nodes
```

### Vendor Examples

**F5 BIG-IP GTM/DNS GSLB**:
```
gtm server DC1-LTM {
    addresses {
        10.1.1.1 {
            device-name dc1-ltm.example.com
        }
    }
    datacenter DC1
    monitor bigip
    virtual-servers {
        /Common/VS-APP {
            destination 192.168.1.100:443
        }
    }
}

gtm pool a POOL-GSLB-APP {
    load-balancing-mode topology
    fallback-mode return-to-dns
    members {
        DC1-LTM:/Common/VS-APP { order 0 }
        DC2-LTM:/Common/VS-APP { order 1 }
    }
    monitor /Common/https_head
    ttl 60
}

gtm wideip a app.example.com {
    pool-lb-mode global-availability
    pools {
        POOL-GSLB-APP { order 0 }
    }
    persistence 3600
}
```

**AWS Route 53 GSLB (Terraform)**:
```hcl
resource "aws_route53_health_check" "dc1" {
  fqdn              = "dc1-app.example.com"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10
}

resource "aws_route53_record" "gslb" {
  zone_id = var.zone_id
  name    = "app.example.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.dc1.dns_name
    zone_id                = aws_lb.dc1.zone_id
    evaluate_target_health = true
  }

  set_identifier  = "dc1-primary"
  health_check_id = aws_route53_health_check.dc1.id
}
```

---

## 12. Monitor Types and Intervals

### Monitor Types Comparison

| Monitor Type | Use Case | Depth | Severity if Missing |
|-------------|----------|-------|---------------------|
| ICMP (ping) | Basic reachability | L3 only | **HIGH** (insufficient alone) |
| TCP connect | Port availability | L4 | **MEDIUM** (does not verify application) |
| HTTP GET | Application health | L7 | Preferred for web applications |
| HTTP POST | Transaction testing | L7 | Best for API backends |
| HTTPS | Encrypted health check | L7 | Required when backend uses TLS |
| Content match | Response validation | L7 | **HIGH** for critical services |
| Script/External | Custom logic | Application | For complex health verification |
| DNS | Name resolution | Application | Required for DNS-dependent services |
| LDAP/RADIUS | Authentication services | Application | Required for auth infrastructure |
| SIP | VoIP services | Application | Required for voice infrastructure |
| MySQL/PostgreSQL | Database availability | Application | Required for database pools |

### Recommended Intervals

| Scenario | Interval | Timeout | Failures | Recovery |
|----------|----------|---------|----------|----------|
| Active health check (standard) | 10s | 5s | 3 | 2 |
| Active health check (fast failover) | 3s | 2s | 3 | 2 |
| GSLB health check | 10-30s | 5-10s | 3 | 2 |
| Passive health check (error-based) | N/A | N/A | 5 errors/30s | 30s |
| DNS server monitoring | 5s | 3s | 3 | 2 |
| Database monitoring | 15s | 10s | 2 | 3 |

---

## Review Procedure Summary

When evaluating DNS and load balancing configurations:

1. **DNS server hardening**: Verify recursion restrictions, version hiding, zone transfer ACLs,
   TSIG authentication, and DNSSEC configuration.
2. **Split-horizon DNS**: Confirm internal records are not exposed externally.
3. **Health checks**: Verify checks test application functionality (not just TCP), intervals
   and timeouts are appropriate, and failure/recovery thresholds prevent flapping.
4. **Persistence**: Confirm the persistence method matches the application requirement and
   does not break load distribution (especially source-IP behind NAT).
5. **SSL/TLS termination**: Verify TLS 1.2+ only, strong cipher suites, valid certificates,
   HSTS, and OCSP stapling.
6. **Connection management**: Confirm connection limits, rate limiting, idle timeouts, and
   Slowloris protections are configured.
7. **GSLB**: Verify health monitoring at each site, appropriate DNS TTLs, failover logic,
   and inter-site authentication.
8. **Classify each finding** using the severity levels in each section.
9. **Prioritize remediation**: CRITICAL findings (open recursive DNS, unrestricted zone
   transfers, exposed private keys) first.

---

## Quick Reference: Severity Summary

| Severity | DNS & Load Balancing Findings |
|----------|-------------------------------|
| CRITICAL | Open recursive DNS to the internet; unrestricted zone transfers (AXFR); private keys in plaintext on load balancer; recursion enabled on authoritative-only server |
| HIGH | DNS not patched; missing DNSSEC validation on resolvers; internal records exposed in external DNS; TCP-only health checks on critical pools; source-IP persistence behind NAT; TLS < 1.2; weak cipher suites; no certificate monitoring; missing HSTS; no connection limits; Slowloris unprotected; GSLB without health monitors; DNS TTL too high for DR; missing fallback pool; SNAT port exhaustion risk; no WAF on public LB |
| MEDIUM | DNS cache misconfiguration; missing NSEC3; no DoT/DoH; health check timeout >= interval; persistence timeout mismatch; no OCSP stapling; no rate limiting; GSLB topology stale; no connection queue limits; missing backend connection reuse; no request smuggling protection; health check endpoint not dedicated; DHCP snooping/DAI prerequisites for DNS security |
| LOW | DNS version string visible; CHAOS queries not disabled; GSLB persistence not configured; missing health check degraded states; DSR not used where beneficial; DNS fallback to plaintext undocumented |
| INFO | Well-configured DNSSEC with automated key rotation; comprehensive health check hierarchy; proper GSLB failover tested; TLS 1.3 with ECDSA certificates; HSTS preload enabled |

---

## Standards References

- **RFC 1034/1035** - Domain Names: Concepts and Implementation
- **RFC 2845** - Secret Key Transaction Authentication for DNS (TSIG)
- **RFC 4033/4034/4035** - DNS Security Introduction and Requirements (DNSSEC)
- **RFC 5011** - Automated Updates of DNS Security Trust Anchors
- **RFC 5936** - DNS Zone Transfer Protocol (AXFR)
- **RFC 7858** - Specification for DNS over Transport Layer Security (DoT)
- **RFC 8484** - DNS Queries over HTTPS (DoH)
- **RFC 8945** - Secret Key Transaction Authentication for DNS (TSIG, updated)
- **RFC 7525** - Recommendations for Secure Use of TLS and DTLS
- **RFC 9325** - Recommendations for Secure Use of TLS and DTLS (updated 7525)
- **NIST SP 800-81-2** - Secure Domain Name System (DNS) Deployment Guide
- **CIS BIND 9 Benchmark** - DNS server hardening
- **CIS F5 BIG-IP Benchmark** - Load balancer security configuration
- **OWASP** - HTTP Request Smuggling, Slowloris, TLS Configuration
- **PCI-DSS v4.0 Requirement 2.2.7** - Encrypt all non-console administrative access using strong cryptography
- **PCI-DSS v4.0 Requirement 4** - Protect cardholder data with strong cryptography during transmission
