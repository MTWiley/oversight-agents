# Capacity Planning

Reference checklist for the `compute-reviewer` agent when evaluating server capacity planning, resource monitoring, sizing, scaling strategy, provisioning automation, and configuration management. This file complements the inline criteria in `review-compute.md` with detailed guidance on utilization tracking, threshold-based alerting, workload sizing, provisioning pipelines, and hardware inventory management.

---

## 1. Current Utilization Monitoring and Baselining

**Category**: Resource Monitoring

You cannot plan capacity for resources you do not measure. Monitoring CPU, memory, disk, and network utilization -- and establishing baselines over meaningful time periods -- is the prerequisite for every other capacity planning activity. Without baselines, alerting thresholds are guesses, sizing is based on vendor marketing, and scaling decisions are reactive rather than proactive.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| CPU utilization is monitored per-server with historical data retention (minimum 90 days) | HIGH | Without CPU history, trend analysis and capacity forecasting are impossible |
| Memory utilization (used, cached, available, swap usage) is monitored | HIGH | Memory exhaustion causes OOM kills and swap thrashing; monitoring prevents surprise outages |
| Disk I/O metrics are monitored (IOPS, throughput MB/s, latency, queue depth) | MEDIUM | Storage bottlenecks manifest as latency; without I/O monitoring the root cause is invisible |
| Disk space utilization is monitored with trend tracking | HIGH | Full filesystems cause application failures, database corruption, and log loss |
| Network interface utilization is monitored (throughput, packet rate, errors, drops) | MEDIUM | Network saturation causes latency spikes and packet drops that are difficult to diagnose without historical data |
| Metrics are collected at appropriate granularity (1-5 minute intervals for capacity planning) | MEDIUM | 15-minute or hourly collection misses short bursts that cause user-visible problems |
| Baselines are established for normal operating ranges (daily, weekly, monthly patterns) | MEDIUM | Without baselines, you cannot distinguish abnormal from seasonal variation |
| Monitoring covers all servers, not just a sample | MEDIUM | Unmonitored servers are invisible to capacity planning |
| Monitoring data is retained long enough for year-over-year trend analysis | LOW | 90 days is minimum; 13 months enables year-over-year comparison for seasonal workloads |
| Monitoring dashboards exist and are accessible to operations and capacity planning teams | LOW | Data that is collected but not visible is data that is not used |
| Per-process/per-service resource consumption is tracked (not just host-level) | MEDIUM | Host-level metrics do not reveal which process is consuming resources |
| GPU utilization is monitored (if applicable: compute %, memory %, temperature) | MEDIUM | GPU workloads (ML training, inference, transcoding) require dedicated monitoring |
| System load average and CPU run queue length are tracked | LOW | Load average provides a high-level view of system pressure beyond CPU percentage |

### Common Mistakes

**Mistake**: Monitoring only CPU percentage without distinguishing user, system, iowait, and steal.

Why it is a problem: A server at 80% CPU utilization could be compute-bound (user), kernel-bound (system), storage-bound (iowait), or hypervisor-contended (steal). Each cause has a different remedy. Aggregated CPU percentage hides the root cause.

```
# BAD - single metric
cpu_utilization: 82%
# What kind of CPU usage? Cannot determine remediation.

# GOOD - broken down by type
cpu_user: 65%      # Application compute -- may need faster cores or more nodes
cpu_system: 8%     # Kernel overhead -- normal range
cpu_iowait: 7%     # Waiting on storage -- investigate disk latency
cpu_steal: 2%      # Hypervisor contention -- investigate noisy neighbor or VM sizing
cpu_idle: 18%
```

**Mistake**: Monitoring memory usage without distinguishing used from cached/buffered.

```
# BAD - misleading metric
memory_used: 95%
# Is this actual application memory or filesystem cache? Very different problems.

# GOOD - detailed memory breakdown
memory_total: 128GB
memory_used: 48GB         # Actual application usage
memory_cached: 72GB       # Filesystem cache (reclaimable)
memory_available: 78GB    # What matters: memory available for new allocations
memory_swap_used: 0GB     # Swap usage indicates real pressure
memory_hugepages: 16GB    # Reserved for database/DPDK
```

**Mistake**: Not tracking disk I/O latency, only throughput.

Why it is a problem: A disk can sustain high throughput with acceptable latency until it hits saturation. At saturation, throughput stays constant but latency spikes exponentially. Without latency monitoring, the transition from healthy to saturated is invisible until applications time out.

```bash
# Key storage metrics to monitor
# IOPS: number of I/O operations per second
# Throughput: MB/s of data transferred
# Latency: time per I/O operation (most important for user experience)
# Queue depth: number of I/O requests waiting (leading indicator of saturation)

# Prometheus node_exporter provides these via:
# node_disk_read_time_seconds_total / node_disk_reads_completed_total  = avg read latency
# node_disk_write_time_seconds_total / node_disk_writes_completed_total = avg write latency
# node_disk_io_time_weighted_seconds_total  = I/O queue time

# Example Prometheus alert for disk latency
groups:
  - name: disk-latency
    rules:
      - alert: HighDiskLatency
        expr: |
          rate(node_disk_read_time_seconds_total[5m])
          / rate(node_disk_reads_completed_total[5m])
          > 0.010
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk read latency > 10ms on {{ $labels.instance }}"
```

### Fix Patterns

**Comprehensive monitoring stack setup**:

```yaml
# Prometheus scrape configuration for server monitoring
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'node-exporter'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'server01:9100'
          - 'server02:9100'
          - 'server03:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.+):9100'
        replacement: '${1}'

  - job_name: 'ipmi-exporter'
    scrape_interval: 60s
    static_configs:
      - targets:
          - 'server01:9290'
          - 'server02:9290'
    params:
      module: [default]
```

**Key Grafana dashboard panels for capacity planning**:

```
# Essential panels for a capacity planning dashboard:

1. CPU Utilization by Type (stacked area: user, system, iowait, steal, idle)
   Query: rate(node_cpu_seconds_total{mode!="idle"}[5m]) * 100

2. Memory Utilization (stacked area: used, cached, available)
   Query: node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

3. Disk Space Utilization with Forecast
   Query: node_filesystem_avail_bytes / node_filesystem_size_bytes * 100
   Add: predict_linear(node_filesystem_avail_bytes{...}[30d], 86400 * 90)

4. Disk I/O Latency (line chart, 95th percentile)
   Query: histogram_quantile(0.95, rate(node_disk_io_time_seconds_total[5m]))

5. Network Throughput (line chart, ingress/egress per interface)
   Query: rate(node_network_receive_bytes_total[5m]) * 8  # bits/sec

6. Swap Usage (should be zero or near-zero in production)
   Query: node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes
```

---

## 2. Threshold Alerting

**Category**: Capacity Alerting

Alerting on resource utilization thresholds provides early warning before capacity exhaustion causes user-visible impact. Alerts should use tiered thresholds (warning and critical) and include context about growth rate, not just current value.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| CPU utilization alerts at warning (80%) and critical (90%) thresholds | MEDIUM | Alert before CPU saturation causes latency degradation |
| Memory utilization alerts at warning (80%) and critical (90%) with swap usage alerting at any non-zero value | HIGH | Memory exhaustion triggers OOM killer, causing unpredictable service failures |
| Disk space alerts at warning (80%) and critical (90%) thresholds | HIGH | Full disk causes application crashes, database corruption, and log loss |
| Disk I/O latency alerts when latency exceeds workload-specific thresholds | MEDIUM | Latency SLA violations indicate storage saturation before throughput metrics show problems |
| Network utilization alerts at warning (70%) and critical (85%) of link capacity | MEDIUM | Network saturation causes packet drops; threshold is lower than CPU/memory due to burst sensitivity |
| Alerts include sustained-duration requirements (e.g., 5 minutes, not instantaneous) | MEDIUM | Instantaneous spikes cause alert fatigue; sustained conditions indicate real problems |
| Predictive alerts exist for disk space (days until full based on growth rate) | MEDIUM | Linear capacity alerts miss slowly-filling volumes; prediction catches them weeks before impact |
| Alert routing reaches the right team (application team vs. infrastructure team) | MEDIUM | Alerts routed to the wrong team are not actionable and create delays |
| Alert fatigue is managed (tuned thresholds, actionable alerts, snooze/acknowledge workflow) | LOW | Too many false alerts causes real alerts to be ignored |
| Alerts are tested (verify they fire when expected) | LOW | Untested alerts may never fire due to misconfigured queries or routing |
| Compound alerts exist for correlated conditions (e.g., high CPU + high disk I/O = I/O-bound workload) | LOW | Compound alerts reduce alert count while improving diagnostic clarity |

### Common Mistakes

**Mistake**: Static threshold alerts without sustained-duration requirements.

```yaml
# BAD - fires on any momentary spike, causes alert fatigue
- alert: HighCPU
  expr: node_cpu_utilization > 0.90
  annotations:
    summary: "High CPU on {{ $labels.instance }}"

# GOOD - requires sustained condition with tiered severity
- alert: HighCPUWarning
  expr: |
    100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    > 80
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "CPU > 80% for 15m on {{ $labels.instance }}"
    runbook: "https://wiki.example.com/runbooks/high-cpu"

- alert: HighCPUCritical
  expr: |
    100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    > 95
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "CPU > 95% for 5m on {{ $labels.instance }}"
    runbook: "https://wiki.example.com/runbooks/high-cpu"
```

**Mistake**: Disk space alerts at fixed percentages without considering absolute size.

Why it is a problem: 90% utilization on a 100GB disk leaves 10GB (may be fine for weeks). 90% utilization on a 10TB disk leaves 1TB (may also be fine). 90% utilization on a 50GB root filesystem leaves 5GB (may fill in hours from logs). Percentage-based thresholds should be combined with absolute minimum thresholds.

```yaml
# GOOD - combined percentage and absolute thresholds
- alert: DiskSpaceLow
  expr: |
    (node_filesystem_avail_bytes{fstype=~"ext4|xfs"}
    / node_filesystem_size_bytes{fstype=~"ext4|xfs"} * 100 < 20)
    or
    (node_filesystem_avail_bytes{fstype=~"ext4|xfs"} < 5e+09)
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Low disk space on {{ $labels.instance }}:{{ $labels.mountpoint }}"

# GOOD - predictive alert: disk will be full within 7 days at current growth rate
- alert: DiskSpacePrediction
  expr: |
    predict_linear(
      node_filesystem_avail_bytes{fstype=~"ext4|xfs"}[7d],
      7 * 24 * 3600
    ) < 0
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} predicted full within 7 days"
```

**Mistake**: No alerting on swap usage.

```yaml
# Swap usage should be zero or near-zero on properly sized servers
# Any persistent swap usage indicates memory pressure

- alert: SwapInUse
  expr: node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes > 0
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Swap in use on {{ $labels.instance }} ({{ $value | humanize1024 }})"
    description: "Persistent swap usage indicates insufficient memory for the workload."
```

---

## 3. Trend Tracking and Forecasting

**Category**: Capacity Forecasting

Trend tracking transforms monitoring data into planning intelligence. By analyzing growth rates and seasonal patterns, teams can predict when capacity additions will be needed and plan procurement and provisioning accordingly, avoiding both emergency purchases and over-provisioning waste.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Resource utilization trends are analyzed at least quarterly | MEDIUM | Quarterly review catches gradual growth before it becomes an emergency |
| Growth rates are calculated for CPU, memory, disk, and network | MEDIUM | Growth rate is the basis for capacity forecasting |
| Seasonal patterns are identified and accounted for (end-of-month, holiday, fiscal year-end) | LOW | Seasonal peaks can be misinterpreted as permanent growth if not analyzed separately |
| Capacity forecasts project at least 6 months into the future | MEDIUM | Hardware procurement lead times require advance planning |
| Forecasts account for planned business growth (new customers, new services, migrations) | MEDIUM | Organic growth trends do not capture step-function changes from business events |
| Capacity reports are shared with stakeholders (application teams, finance, management) | LOW | Capacity planning is a cross-functional activity; siloed reports reduce effectiveness |
| Historical incidents caused by capacity exhaustion are tracked and feed back into planning | LOW | Post-incident capacity additions are reactive; feeding incidents into the forecast process prevents recurrence |

### Common Mistakes

**Mistake**: Linear extrapolation of growth without considering seasonal patterns.

Why it is a problem: A server that uses 60% CPU in January and 80% in March might appear to be growing at 10% per month. But if March is always the busiest month (quarter-end processing), the actual year-over-year growth might be only 5%. Without seasonal decomposition, capacity is over-provisioned.

**Mistake**: Not accounting for planned business events in capacity forecasts.

```
# Example: Organic growth shows 5% monthly CPU increase
# But the business is planning:
# - Q2: Migration of 500 users from legacy system (estimated +20% CPU)
# - Q3: Launch of new analytics feature (estimated +15% memory)
# - Q4: Holiday traffic (historical +40% over baseline)

# Without business context, the capacity plan would predict:
# Month 1-6: Linear growth at 5%/month -> inadequate by Q3

# With business context, the capacity plan should show:
# Q1: 5%/month organic growth
# Q2: +20% step increase from migration
# Q3: +15% memory increase from new feature
# Q4: +40% seasonal peak requiring temporary or burst capacity
```

### Fix Patterns

**Capacity forecasting with Prometheus and recording rules**:

```yaml
# Recording rules for capacity trending
groups:
  - name: capacity-trending
    interval: 1h
    rules:
      # Average CPU utilization per server (hourly)
      - record: capacity:cpu_utilization:avg1h
        expr: |
          100 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[1h]) * 100)

      # Memory utilization percentage (hourly)
      - record: capacity:memory_utilization:avg1h
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

      # Disk usage percentage (hourly)
      - record: capacity:disk_utilization:avg1h
        expr: |
          (1 - node_filesystem_avail_bytes{fstype=~"ext4|xfs"}
          / node_filesystem_size_bytes{fstype=~"ext4|xfs"}) * 100

      # 30-day linear growth rate (percentage points per day)
      - record: capacity:cpu_growth_rate:30d
        expr: |
          deriv(capacity:cpu_utilization:avg1h[30d]) * 86400

      - record: capacity:disk_growth_rate:30d
        expr: |
          deriv(capacity:disk_utilization:avg1h[30d]) * 86400
```

**Quarterly capacity report template**:

```
# Quarterly Capacity Review - Q1 2026
# Generated: 2026-03-31

## Executive Summary
- Fleet: 150 servers across 3 data centers
- Current average utilization: CPU 62%, Memory 71%, Disk 55%
- Projected capacity exhaustion: 0 servers within 6 months at current growth
- Action items: 12 servers recommended for memory upgrade, 5 for disk expansion

## Resource Utilization Summary
| Resource | P50 Utilization | P90 Utilization | P99 Utilization | Growth Rate (monthly) |
|----------|----------------|-----------------|-----------------|----------------------|
| CPU      | 45%            | 78%             | 92%             | +2.1%                |
| Memory   | 58%            | 85%             | 94%             | +3.4%                |
| Disk     | 42%            | 72%             | 88%             | +1.8%                |
| Network  | 12%            | 35%             | 67%             | +1.2%                |

## Servers Approaching Thresholds
| Server      | Resource | Current | Growth Rate | Projected 80% Date | Projected 90% Date |
|-------------|----------|---------|-------------|--------------------|--------------------|
| db-prod-01  | Memory   | 87%     | +1.2%/mo    | EXCEEDED           | 2026-05-15         |
| app-prod-12 | Disk     | 82%     | +3.5%/mo    | EXCEEDED           | 2026-04-22         |
| batch-03    | CPU      | 76%     | +4.1%/mo    | 2026-04-15         | 2026-06-10         |

## Recommendations
1. db-prod-01: Add 128GB memory (from 256GB to 384GB) - Q2 maintenance window
2. app-prod-12: Expand /data volume by 2TB - immediate, online expansion possible
3. batch-03: Evaluate workload optimization before adding CPU (possible inefficiency)
```

---

## 4. Server Sizing

**Category**: Server Sizing

Proper server sizing matches hardware resources to workload requirements. Undersized servers cause performance problems and outages. Oversized servers waste capital and power budget. Sizing decisions should be based on measured workload characteristics, not vendor recommendations or guesswork.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| CPU core count matches workload parallelism (not just "more is better") | MEDIUM | An 8-thread application on a 64-core server wastes 87% of CPU investment |
| CPU clock speed vs. core count tradeoff is evaluated for the workload type | MEDIUM | Single-threaded workloads (many databases) benefit from higher clock speed; parallel workloads benefit from more cores |
| Memory is sized for working set, not just total data size | HIGH | Memory should hold the working set (frequently accessed data); total data set lives on storage |
| Memory channels are fully populated for maximum bandwidth | LOW | Partially populated memory channels reduce bandwidth proportionally |
| Storage IOPS and latency match workload requirements (not just capacity) | HIGH | A 10TB HDD array with 200 IOPS cannot serve a workload that needs 50,000 IOPS |
| Storage type matches workload pattern (NVMe for latency-sensitive, SSD for mixed, HDD for sequential/archival) | MEDIUM | Wrong storage type wastes money (NVMe for archive) or causes bottlenecks (HDD for OLTP) |
| Network bandwidth matches data transfer requirements | MEDIUM | A 1GbE NIC on a server transferring 10TB/day creates a bottleneck |
| Network interface count and bonding configuration match availability requirements | MEDIUM | Single NIC is a single point of failure for network connectivity |
| Server sizing is documented with rationale (why these specs were chosen) | LOW | Undocumented sizing decisions cannot be validated or improved for future deployments |
| Sizing accounts for OS and system overhead (reserve 10-15% of resources) | MEDIUM | The OS, monitoring agents, and system processes consume resources that must not be allocated to applications |
| Sizing is validated against actual utilization after deployment (right-sizing feedback loop) | MEDIUM | Initial sizing estimates should be compared against reality to improve future sizing |

### Common Mistakes

**Mistake**: Sizing CPU by core count without considering clock speed and workload parallelism.

```
# BAD - more cores always better?
Workload: Single-threaded database engine (e.g., Redis, single-threaded queries)
Server: 2x AMD EPYC 7763 (64 cores each, 2.45 GHz base)
Result: 128 cores but only 1 is busy; the 2.45 GHz clock limits single-thread performance

# GOOD - match CPU to workload characteristics
Workload: Single-threaded database engine
Server: 1x Intel Xeon w9-3495X (56 cores, 4.8 GHz turbo)
Result: Fewer cores but faster single-thread performance for the actual workload

# For parallel workloads (compilation, rendering, many-container hosts):
# Optimize for core count over clock speed
# AMD EPYC 9654 (96 cores) or Intel Xeon 6980P (128 cores)
```

**Mistake**: Sizing memory based on total database size, not working set.

```
# BAD - total database is 2TB, so server needs 2TB RAM
Database size: 2TB
Server memory: 2TB
Cost: ~$40,000 for memory alone

# GOOD - analyze working set size
Database size: 2TB
Working set (frequently accessed data): 200GB
Index size: 80GB
Server memory: 384GB (covers working set + indexes + OS overhead + page cache)
Cost: ~$6,000 for memory

# How to measure working set:
# PostgreSQL: pg_stat_user_tables -> seq_scan, idx_scan patterns
# MongoDB: db.serverStatus().wiredTiger.cache
# Linux: /proc/meminfo -> Active(anon) + Active(file)
# vmstat: look at si/so (swap in/out) -- if zero, working set fits in memory
```

**Mistake**: Choosing storage based on capacity alone, ignoring IOPS and latency.

```
# BAD - 10TB needed, cheapest option selected
Storage: 10x 1TB 7200 RPM SATA HDD (RAID 10)
Capacity: 5TB usable (RAID 10 overhead)
Random IOPS: ~1,000 (100 IOPS per spindle in RAID 10)
Latency: 5-10ms average

Workload requirement: 50,000 random IOPS, < 1ms latency
Result: 50x slower than needed; application timeouts, user complaints

# GOOD - size for IOPS and latency first, then capacity
Storage: 4x 1.6TB NVMe SSD (RAID 10 or RAID 5 depending on write ratio)
Capacity: 3.2-4.8TB usable
Random IOPS: 400,000+ (100,000+ per drive)
Latency: 0.1-0.2ms average
Result: Exceeds IOPS requirement with room for growth
```

### Fix Patterns

**Server sizing worksheet**:

```
# Server Sizing Worksheet - [Application Name]
# Date: YYYY-MM-DD
# Author: [Name]
# Purpose: [New deployment / capacity upgrade / refresh]

## Workload Characteristics
- Application type: [Database / Web / Batch / ML Training / General]
- Concurrency model: [Multi-threaded / Multi-process / Event-loop / Single-threaded]
- Maximum concurrent threads/processes: [N]
- Memory access pattern: [Working set size / Sequential / Random]
- I/O pattern: [Random read / Sequential write / Mixed]
- Network pattern: [Request-response / Streaming / Bulk transfer]
- Availability requirement: [99.9% / 99.99% / 99.999%]

## CPU Sizing
- Measured/estimated CPU requirement: [N] cores at [X] GHz
- Parallelism: [Application uses N threads effectively]
- Clock speed sensitivity: [High (single-threaded) / Low (parallel)]
- Recommendation: [CPU model, socket count]
- Overhead reserve: 15% (for OS, monitoring, system processes)
- Final: [N cores] at [X GHz base / Y GHz turbo]

## Memory Sizing
- Working set size: [N GB] (measured or estimated)
- Application heap/stack: [N GB]
- OS page cache (for I/O workloads): [N GB]
- OS and system overhead: [4-8 GB]
- Hugepage reservation (if applicable): [N GB]
- Total recommended: [N GB]
- Overhead reserve: 10%
- Final: [N GB] (next standard DIMM configuration)

## Storage Sizing
- Capacity requirement: [N TB] (current + 12 months growth)
- IOPS requirement: [N] random read, [N] random write
- Throughput requirement: [N MB/s] sequential read, [N MB/s] sequential write
- Latency requirement: < [N ms] P99
- Storage type: [NVMe / SSD / HDD / Hybrid]
- RAID level: [0 / 1 / 5 / 6 / 10]
- Final: [N x size type] in RAID [level]

## Network Sizing
- Bandwidth requirement: [N Gbps] sustained
- Packet rate: [N Kpps]
- Redundancy: [Active-backup / LACP / Separate subnets]
- Final: [N x speed] NIC(s) with [bonding mode]
```

---

## 5. Scaling Strategy

**Category**: Scaling Strategy

Scaling strategy defines how capacity is added when current resources are insufficient. The choice between vertical scaling (bigger servers), horizontal scaling (more servers), and hybrid approaches depends on the application architecture, budget constraints, and availability requirements.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Scaling strategy is defined and documented (vertical, horizontal, or hybrid) | MEDIUM | Without a strategy, scaling is ad-hoc and reactive |
| Vertical scaling limits are understood (maximum CPU, memory, storage for the platform) | MEDIUM | Every server has a maximum configuration; planning must account for the ceiling |
| Horizontal scaling is architecturally supported (stateless or externalized state) | HIGH | Adding servers to a horizontally-scalable tier requires stateless design; discovering this during an outage is too late |
| Cluster sizing accounts for N+1 or N+2 redundancy | HIGH | A cluster that is 100% utilized cannot tolerate a node failure without performance degradation |
| Auto-scaling policies are defined for elastic workloads (cloud/virtualized environments) | MEDIUM | Manual scaling for variable workloads either wastes resources (over-provisioned) or risks outages (under-provisioned) |
| Scale-up triggers are defined (what metric, what threshold, what action) | MEDIUM | Undefined triggers mean scaling is initiated by someone noticing a problem, which is too late |
| Scale-down procedures exist to reclaim unused capacity | LOW | Resources added during peak periods should be reclaimed; otherwise cost grows monotonically |
| Instance right-sizing reviews are conducted periodically (quarterly) | MEDIUM | Initial sizing is estimated; actual utilization often differs significantly |
| Spot/preemptible instances are evaluated for fault-tolerant batch workloads | LOW | Spot instances provide 60-90% cost savings for workloads that can tolerate interruption |
| Scaling tests have been performed (verify the application scales as expected) | MEDIUM | Assumed scalability must be validated; many applications have unexpected bottlenecks |
| Lead time for scaling is documented (how long does it take to add a server or node) | MEDIUM | If adding a bare-metal server takes 6 weeks, the scaling trigger must fire 6 weeks before capacity is needed |

### Common Mistakes

**Mistake**: Planning for horizontal scaling without verifying the application is stateless.

Why it is a problem: Adding a second web server behind a load balancer does not help if the application stores session state in local memory, writes files to local disk, or uses in-process caches that must be consistent. The application will produce incorrect results, not just performance problems.

```
# BAD - assumed horizontal scalability
Application stores sessions in local memory -> added second server
Result: Users randomly lose sessions when load balancer routes to the other server

# GOOD - verify statelessness before horizontal scaling
Checklist before adding servers to a tier:
[ ] Sessions externalized to Redis/database
[ ] File uploads stored on shared storage (NFS, S3, object store)
[ ] Caches are either local-only (acceptable for hit rate) or distributed
[ ] No server-specific configuration (all servers are identical)
[ ] Health checks are application-level (not just TCP port)
[ ] Graceful shutdown drains in-flight requests
```

**Mistake**: N+0 cluster sizing with no failure tolerance.

```
# BAD - 3-node cluster sized for exactly 3 nodes of capacity
Cluster: 3 nodes, each at 90% CPU utilization
Node failure: remaining 2 nodes at 135% -> overloaded, cascading failure

# GOOD - N+1 cluster sizing
Cluster: 4 nodes, each at 67% CPU utilization (total workload = 3 nodes)
Node failure: remaining 3 nodes at 89% -> degraded but functional

# BETTER - N+2 for critical workloads
Cluster: 5 nodes, each at 60% CPU utilization (total workload = 3 nodes)
Two-node failure: remaining 3 nodes at 100% -> at capacity but functional
```

**Mistake**: No scale-down procedure, leading to cost creep.

```
# Common pattern: scale up during incident, never scale back down
# January: 10 servers at $5,000/month
# March incident: scaled to 15 servers, $7,500/month
# June: incident is long resolved, still running 15 servers
# September: another incident, scaled to 20 servers, $10,000/month
# December: still running 20 servers, actual need is 12

# FIX: scheduled right-sizing reviews
# - Monthly review of over-provisioned resources
# - Automated recommendations based on P95 utilization
# - Scale-down requires the same approval process as scale-up (not harder)
```

### Fix Patterns

**Auto-scaling policy example (Kubernetes HPA)**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
```

---

## 6. Resource Reservation

**Category**: Resource Reservation

Resource reservation ensures that critical system functions, burst handling, and overhead have guaranteed resources. Without reservations, a server at 100% utilization has no headroom for system processes, monitoring agents, or sudden load spikes.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| System overhead is reserved (10-15% of CPU and memory not allocated to applications) | MEDIUM | The OS kernel, sshd, monitoring agents, and system processes need resources |
| Burst capacity headroom exists (20-30% of peak for unexpected spikes) | MEDIUM | Servers sized for average load fail during peak; servers sized for peak waste resources during average |
| Filesystem page cache is accounted for in memory sizing | MEDIUM | The Linux page cache uses "available" memory for I/O performance; allocating all memory to applications eliminates page cache benefit |
| I/O reservations exist for critical workloads (cgroups blkio, I/O scheduling priority) | LOW | Without I/O reservation, a batch job can starve the database of I/O |
| CPU reservations exist for critical services (cgroups, cpuset, systemd CPUQuota) | MEDIUM | Prevents non-critical services from starving critical ones during contention |
| Memory reservations prevent OOM kills of critical services (MemoryLow, oom_score_adj) | HIGH | The OOM killer may kill the database instead of the log collector if not guided |
| Kubernetes resource requests are set accurately (not just copied from limits) | MEDIUM | Over-requested resources waste cluster capacity; under-requested resources cause scheduling issues |
| Kubernetes resource limits are set to prevent unbounded growth | HIGH | Missing limits allow a single pod to consume all node resources |
| Overcommit ratios are defined and monitored for virtualized environments | MEDIUM | CPU overcommit of 4:1 is common; memory overcommit above 1.5:1 risks ballooning and swapping |

### Common Mistakes

**Mistake**: Allocating 100% of server memory to the application, leaving nothing for page cache or OS.

```
# BAD - server has 128GB, application configured to use 128GB
Server memory: 128GB
JVM heap: -Xmx124G (leaving only 4GB for OS)
Result: No page cache, excessive disk I/O, poor performance for file-based operations

# GOOD - reserve memory for OS and page cache
Server memory: 128GB
JVM heap: -Xmx96G (75% of total)
OS + overhead: ~4GB
Page cache: ~28GB available for filesystem caching
Result: File reads served from page cache, reduced disk I/O
```

**Mistake**: Not using oom_score_adj to protect critical services.

```bash
# BAD - OOM killer selects the largest process (often the database)
# When memory is exhausted, the database is killed instead of the log collector

# GOOD - protect critical services from OOM killer
# /etc/systemd/system/postgresql.service.d/oom.conf
[Service]
OOMScoreAdjust=-900  # Strongly prefer NOT to kill

# /etc/systemd/system/log-collector.service.d/oom.conf
[Service]
OOMScoreAdjust=500   # OK to kill if needed

# Or via cgroups v2 memory protection
# /etc/systemd/system/postgresql.service.d/memory.conf
[Service]
MemoryLow=48G        # Kernel will try to avoid reclaiming below this
MemoryHigh=56G       # Throttle allocations above this
MemoryMax=64G        # Hard limit, OOM kill above this
```

**Mistake**: Kubernetes requests and limits not set or set incorrectly.

```yaml
# BAD - no requests or limits
containers:
  - name: app
    image: myapp:1.0
    # Result: pod can consume all node resources; scheduler cannot make informed decisions

# BAD - requests equal to limits (no burst capacity)
containers:
  - name: app
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
    # Result: guaranteed QoS but no ability to burst; wastes resources during idle

# GOOD - requests reflect typical usage, limits cap burst
containers:
  - name: app
    resources:
      requests:
        cpu: "1"          # Typical usage
        memory: "4Gi"     # Typical usage
      limits:
        cpu: "4"          # Burst up to 4 cores
        memory: "8Gi"     # Hard limit to prevent OOM impact on node
    # Result: Burstable QoS, efficient resource sharing, bounded impact
```

---

## 7. Provisioning Automation

**Category**: Provisioning Automation

Manual server provisioning is slow, error-prone, and unreproducible. Automated provisioning via PXE boot, kickstart/preseed/autoinstall, and orchestration tools ensures consistent, documented, and rapid server deployment. The provisioning pipeline is the first step in configuration drift prevention.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Server provisioning is automated (not manual OS installation from USB/ISO) | MEDIUM | Manual installation takes hours per server and produces inconsistent results |
| PXE boot infrastructure exists for bare-metal provisioning | MEDIUM | PXE enables network-based OS installation without physical media |
| Kickstart (RHEL), preseed (Debian), or autoinstall (Ubuntu) templates are version-controlled | HIGH | Unversioned templates drift over time and cannot be audited or reproduced |
| Provisioning templates include disk layout, partitioning, and LVM configuration | MEDIUM | Manual disk layout creates inconsistency across servers |
| Provisioning templates include network configuration (bonding, VLAN tagging) | MEDIUM | Manual network configuration is error-prone and delays deployment |
| Provisioning includes initial security hardening (SSH keys, disable root password, firewall rules) | HIGH | A newly provisioned server with default credentials is vulnerable from the moment it boots |
| Provisioning integrates with CMDB/inventory system for automatic registration | LOW | Manual inventory updates are forgotten; automated registration ensures accuracy |
| Provisioning pipeline includes post-installation validation (OS version, network, disk, packages) | MEDIUM | A provisioned server that fails validation should be caught before entering production |
| Provisioning supports different server roles (database, web, compute) with role-specific templates | LOW | One-size-fits-all provisioning wastes time customizing after installation |
| Provisioning time from power-on to production-ready is measured and optimized | LOW | Slow provisioning delays scaling response time |
| Foreman, MAAS, or OpenStack Ironic is used for fleet-scale bare-metal management | MEDIUM | Ad-hoc PXE scripts do not scale to hundreds of servers |
| Cloud-init or ignition is used for first-boot configuration in cloud/virtualized environments | MEDIUM | Manual first-boot configuration prevents infrastructure-as-code practices |

### Common Mistakes

**Mistake**: Provisioning from a manually maintained kickstart file on an engineer's laptop.

Why it is a problem: The kickstart file is not version-controlled, not reviewed, and not reproducible. When the engineer leaves or the laptop is replaced, the provisioning recipe is lost. Different engineers use different kickstart files, creating fleet inconsistency.

**Mistake**: Provisioning without initial security hardening.

```
# BAD kickstart snippet - default root password, no SSH key, no firewall
rootpw --plaintext changeme
firewall --disabled

# GOOD kickstart snippet - locked root, SSH key, firewall enabled
rootpw --lock
user --name=deploy --groups=wheel --lock
sshkey --username=deploy "ssh-ed25519 AAAA... deploy@provisioning"
firewall --enabled --ssh
```

### Fix Patterns

**RHEL/CentOS kickstart template (version-controlled)**:

```
# /srv/provisioning/kickstart/rhel9-base.ks
# Version: 2.1.0
# Last modified: 2026-01-15
# Purpose: Base RHEL 9 installation for all server roles

# System configuration
lang en_US.UTF-8
keyboard us
timezone UTC --utc
rootpw --lock
user --name=ansible --groups=wheel --lock
sshkey --username=ansible "ssh-ed25519 AAAA... ansible@provisioning"

# Network configuration (bonded interfaces)
network --bootproto=dhcp --device=bond0 --bondslaves=eno1,eno2 \
  --bondopts=mode=802.3ad,miimon=100,lacp_rate=fast \
  --activate --onboot=yes
network --bootproto=static --device=eno3 --ip=172.16.0.X \
  --netmask=255.255.255.0 --gateway=172.16.0.1 \
  --nameserver=10.0.0.53 --nodefroute --activate --onboot=yes

# Disk configuration
zerombr
clearpart --all --initlabel
part /boot/efi --fstype=efi --size=600
part /boot --fstype=xfs --size=1024
part pv.01 --size=1 --grow
volgroup vg_root pv.01
logvol / --vgname=vg_root --fstype=xfs --size=20480 --name=lv_root
logvol /var --vgname=vg_root --fstype=xfs --size=20480 --name=lv_var
logvol /var/log --vgname=vg_root --fstype=xfs --size=10240 --name=lv_log
logvol /tmp --vgname=vg_root --fstype=xfs --size=5120 --name=lv_tmp \
  --fsoptions="nodev,nosuid,noexec"
logvol swap --vgname=vg_root --size=8192 --name=lv_swap

# Package selection
%packages
@^minimal-environment
chrony
tuned
openssh-server
python3
microcode_ctl
openscap-scanner
scap-security-guide
-telnet
-rsh
-tftp
%end

# Post-installation
%post
# Enable firewall
systemctl enable firewalld
firewall-cmd --permanent --add-service=ssh

# Harden SSH
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Enable tuned
systemctl enable tuned

# Register with Foreman/Satellite
curl -sS https://foreman.example.com/register | bash

# Signal provisioning system that installation is complete
curl -X POST https://provisioning.example.com/api/servers/$(hostname)/status \
  -d '{"status": "provisioned", "timestamp": "'$(date -Iseconds)'"}'
%end

# Reboot after installation
reboot --eject
```

**Ubuntu autoinstall template**:

```yaml
# /srv/provisioning/autoinstall/ubuntu-base.yml
#cloud-config
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard:
    layout: us
  identity:
    hostname: ubuntu-server
    username: deploy
    password: "!"  # Locked account
  ssh:
    install-server: true
    authorized-keys:
      - ssh-ed25519 AAAA... ansible@provisioning
    allow-pw: false
  storage:
    layout:
      name: lvm
      sizing-policy: scaled
  packages:
    - python3
    - chrony
    - tuned
    - linux-firmware
  late-commands:
    - curtin in-target -- systemctl enable ufw
    - curtin in-target -- ufw allow ssh
    - curtin in-target -- systemctl enable tuned
```

**Foreman/Satellite integration for fleet provisioning**:

```bash
# Foreman provisioning workflow:
# 1. Define host group with OS, partition table, and provisioning template
# 2. Server PXE boots and contacts Foreman via DHCP/TFTP
# 3. Foreman serves kickstart/preseed based on host group
# 4. Server installs OS and registers with Foreman
# 5. Puppet/Ansible applies configuration management

# Foreman API: provision a new server
curl -X POST https://foreman.example.com/api/hosts \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "host": {
      "name": "web-prod-06",
      "hostgroup_id": 5,
      "location_id": 1,
      "organization_id": 1,
      "build": true,
      "mac": "aa:bb:cc:dd:ee:ff",
      "ip": "10.0.1.56",
      "interfaces_attributes": [{
        "mac": "aa:bb:cc:dd:ee:ff",
        "ip": "10.0.1.56",
        "type": "Nic::Managed",
        "provision": true
      }]
    }
  }'
```

---

## 8. Configuration Management

**Category**: Configuration Management

Configuration management ensures that servers remain in their desired state after provisioning. Without it, configuration drift accumulates through manual changes, emergency fixes, and ad-hoc modifications, eventually creating unique "snowflake" servers that cannot be reproduced or audited.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A configuration management tool is in use (Ansible, Puppet, Chef, Salt) | MEDIUM | Without CM, every server diverges from the baseline over time |
| Configuration is stored in version control (Git) | HIGH | Unversioned configuration cannot be audited, reviewed, or rolled back |
| Configuration changes go through code review (pull request workflow) | MEDIUM | Unreviewed configuration changes bypass the team's expertise and audit trail |
| Configuration management runs are scheduled (not just ad-hoc) | MEDIUM | Ad-hoc runs mean drift accumulates between runs; scheduled runs enforce convergence |
| Drift detection alerts when servers deviate from desired state | MEDIUM | Without drift detection, unauthorized manual changes go unnoticed |
| Configuration is idempotent (running it twice produces the same result) | MEDIUM | Non-idempotent configuration causes errors on re-run and prevents scheduled enforcement |
| Secrets are managed separately from configuration (Vault, sealed secrets, Ansible Vault) | HIGH | Secrets in plaintext configuration files are exposed in version control |
| Role/profile separation exists (reusable roles for different server types) | LOW | Monolithic configuration files are hard to maintain and reuse |
| Configuration management covers all server aspects (packages, services, files, cron, users, firewall) | MEDIUM | Partial CM leaves unmanaged aspects vulnerable to drift |
| Configuration changes are tested before production deployment (molecule, kitchen, staging) | MEDIUM | Untested configuration changes can break production servers |
| Emergency manual changes are documented and backported to configuration management | MEDIUM | Manual fixes that are not backported create permanent drift |
| Configuration management agent is monitored (last run time, success/failure) | MEDIUM | A failed or stalled CM agent means the server is no longer being managed |

### Common Mistakes

**Mistake**: Using Ansible in ad-hoc mode without scheduled runs.

Why it is a problem: Ansible is often used as a "push on demand" tool rather than a continuous enforcement mechanism. When an engineer makes a manual change on a server, there is no process to detect or correct the drift unless Ansible is run again -- which may not happen for months.

```bash
# BAD - ad-hoc, inconsistent usage
ssh server01 "sysctl -w vm.swappiness=10"  # Manual change, no CM
ansible-playbook site.yml  # Run whenever someone remembers

# GOOD - scheduled, enforced, drift-detecting
# Ansible pull or AWX/Tower scheduled job
# /etc/cron.d/ansible-pull
0 */4 * * * root ansible-pull -U https://git.example.com/ansible/server-config.git \
  -i localhost, -e "ansible_connection=local" site.yml \
  >> /var/log/ansible-pull.log 2>&1

# Or via AWX/Ansible Automation Platform:
# - Schedule: Every 4 hours
# - Mode: Check + Apply
# - Notification: Alert on changes (indicates drift was corrected)
```

**Mistake**: Secrets stored in Ansible playbooks or variable files committed to Git.

```yaml
# BAD - secret in plain text in version control
# group_vars/database.yml
db_password: "SuperSecret123!"
idrac_password: "calvin"

# GOOD - secrets managed via Ansible Vault
# Encrypt the file:
# ansible-vault encrypt group_vars/database.yml

# Or reference HashiCorp Vault:
# group_vars/database.yml
db_password: "{{ lookup('community.hashi_vault.hashi_vault',
                  'secret/data/database:password') }}"
idrac_password: "{{ lookup('community.hashi_vault.hashi_vault',
                    'secret/data/idrac/server01:password') }}"
```

**Mistake**: Configuration management that is not idempotent.

```yaml
# BAD - appends to file on every run (not idempotent)
- name: Add sysctl setting
  ansible.builtin.shell: echo "vm.swappiness = 10" >> /etc/sysctl.conf

# GOOD - idempotent approach
- name: Set sysctl parameters
  ansible.posix.sysctl:
    name: vm.swappiness
    value: "10"
    state: present
    sysctl_file: /etc/sysctl.d/99-tuning.conf
    reload: true
```

### Fix Patterns

**Drift detection with Ansible**:

```yaml
# drift-check.yml - Run in check mode to detect drift
# ansible-playbook drift-check.yml --check --diff

---
- hosts: all
  tasks:
    - name: Check all managed files for drift
      ansible.builtin.include_role:
        name: "{{ item }}"
      loop:
        - base-os
        - security-hardening
        - monitoring-agent
        - server-tuning

# AWX job template configuration:
# - Playbook: drift-check.yml
# - Job Type: Check (--check --diff)
# - Schedule: Daily at 06:00
# - Notification: Webhook to Slack/Teams on any changes detected
```

---

## 9. Hardware Inventory Tracking

**Category**: Hardware Inventory

A comprehensive hardware inventory is the foundation for capacity planning, procurement, warranty management, and incident response. Without accurate inventory, teams cannot determine what hardware is available, what needs replacement, or what is affected by a vendor recall.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All physical servers are tracked in a CMDB or inventory system | MEDIUM | Untracked servers are invisible to capacity planning, patching, and security |
| Inventory includes make, model, serial number, and service tag | MEDIUM | Required for vendor support cases, warranty claims, and recall identification |
| Inventory includes physical location (data center, row, rack, U position) | MEDIUM | Physical location is needed for maintenance, power capacity planning, and cooling |
| Inventory includes hardware specifications (CPU, memory, disk, NIC) | MEDIUM | Specifications are needed for workload placement and capacity calculations |
| Inventory includes server role and application assignment | MEDIUM | Mapping servers to applications enables impact assessment during outages |
| Inventory includes network connections (switch port, VLAN, IP addresses) | MEDIUM | Network topology mapping is needed for troubleshooting and migration planning |
| Inventory includes power connections (PDU, circuit, power draw) | LOW | Power capacity planning requires per-server power consumption data |
| Inventory is automatically updated (not manual data entry) | MEDIUM | Manual inventory is always incomplete and outdated |
| Inventory reconciliation is performed regularly (physical audit vs. CMDB) | LOW | Discrepancies between physical reality and inventory indicate process failures |
| Inventory supports querying (how many servers with X CPU, Y memory, in Z location) | LOW | Ad-hoc queries are needed for capacity planning and procurement |
| Inventory tracks lifecycle status (active, maintenance, decommissioning, decommissioned) | MEDIUM | Servers in different lifecycle stages have different management requirements |

### Common Mistakes

**Mistake**: Inventory maintained in a spreadsheet that is always out of date.

Why it is a problem: Spreadsheets lack access control, audit trail, API access, and automation integration. When a server is added, moved, or decommissioned, the spreadsheet update is a manual step that is frequently forgotten. After a year, the spreadsheet matches reality perhaps 70% of the time.

**Mistake**: Inventory does not include network topology.

Why it is a problem: When a switch fails, knowing which servers are connected to it requires walking into the data center and tracing cables -- unless the inventory includes switch port assignments. This turns a 5-minute impact assessment into a 30-minute physical audit during an outage.

### Fix Patterns

**Automated inventory collection via Ansible facts**:

```yaml
# inventory-collection.yml
---
- hosts: all
  gather_facts: true
  tasks:
    - name: Collect hardware facts
      ansible.builtin.setup:
        gather_subset:
          - hardware
          - network
          - virtual

    - name: Collect BMC information via Redfish
      community.general.redfish_info:
        baseuri: "{{ bmc_address }}"
        username: "{{ bmc_user }}"
        password: "{{ bmc_password }}"
        category: Systems
        command: GetSystemInventory
      register: system_info
      delegate_to: localhost

    - name: Update inventory database
      ansible.builtin.uri:
        url: "https://cmdb.example.com/api/servers/{{ inventory_hostname }}"
        method: PUT
        headers:
          Content-Type: application/json
          Authorization: "Bearer {{ cmdb_token }}"
        body_format: json
        body:
          hostname: "{{ inventory_hostname }}"
          fqdn: "{{ ansible_fqdn }}"
          os: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
          kernel: "{{ ansible_kernel }}"
          cpu_model: "{{ ansible_processor[2] }}"
          cpu_count: "{{ ansible_processor_vcpus }}"
          memory_mb: "{{ ansible_memtotal_mb }}"
          disk_total_gb: "{{ ansible_mounts | map(attribute='size_total') | map('int') | sum // (1024**3) }}"
          network_interfaces: "{{ ansible_interfaces }}"
          serial_number: "{{ ansible_product_serial }}"
          manufacturer: "{{ ansible_system_vendor }}"
          model: "{{ ansible_product_name }}"
          virtualization_role: "{{ ansible_virtualization_role }}"
          virtualization_type: "{{ ansible_virtualization_type }}"
          last_inventory_date: "{{ ansible_date_time.iso8601 }}"
      delegate_to: localhost
```

**NetBox as infrastructure inventory (CMDB)**:

```bash
# NetBox provides a comprehensive CMDB with API access
# Install via Docker Compose: https://github.com/netbox-community/netbox-docker

# API: Add a server to NetBox
curl -X POST https://netbox.example.com/api/dcim/devices/ \
  -H "Authorization: Token ${NETBOX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "web-prod-06",
    "device_type": {"slug": "poweredge-r750"},
    "role": {"slug": "web-server"},
    "site": {"slug": "dc1"},
    "rack": {"name": "Row3-Rack12"},
    "position": 20,
    "face": "front",
    "serial": "ABC1234",
    "asset_tag": "ASSET-1234",
    "status": "active",
    "tenant": {"slug": "infrastructure"},
    "tags": [{"slug": "production"}, {"slug": "web-tier"}]
  }'

# Query: Find all servers with less than 128GB memory in DC1
curl "https://netbox.example.com/api/dcim/devices/?site=dc1&role=server" \
  -H "Authorization: Token ${NETBOX_TOKEN}" | jq '.results[]'
```

---

## Quick Reference: Severity Summary

| Severity | Capacity Planning Findings |
|----------|----------------------------|
| CRITICAL | (None -- capacity planning findings are rarely immediately exploitable; they compound over time into HIGH when ignored) |
| HIGH | No CPU/memory/disk monitoring; no disk space alerting; memory exhaustion without OOM protection; horizontal scaling assumed but application is stateful; cluster sized at N+0 with no failure tolerance; provisioning templates with default credentials; secrets in configuration management; Kubernetes pods without memory limits |
| MEDIUM | No disk I/O monitoring; no network monitoring; insufficient metric granularity; no baselines established; no monitoring on some servers; no per-process metrics; CPU governor wrong for workload; no trend analysis; no capacity forecasting; sizing not validated post-deployment; scaling strategy undefined; vertical limits not understood; auto-scaling not defined; scale triggers undefined; instance right-sizing not reviewed; scaling not tested; scaling lead time undocumented; system overhead not reserved; burst capacity not reserved; page cache not accounted for; CPU reservations missing; overcommit ratios undefined; Kubernetes requests not set accurately; provisioning not automated; no PXE infrastructure; provisioning templates not covering disk/network; no post-provision validation; no configuration management; drift detection missing; CM not idempotent; CM not tested; emergency changes not backported; CM agent not monitored; hardware inventory incomplete or manual; lifecycle status not tracked |
| LOW | Monitoring retention under 13 months; no dashboards; system load average not tracked; alert fatigue not managed; alerts not tested; compound alerts missing; seasonal patterns not identified; forecasts not shared; capacity reports not accessible; memory channels not fully populated; server sizing not documented; provisioning not integrated with CMDB; role-specific provisioning templates missing; provisioning speed not measured; CM role separation missing; inventory reconciliation not performed; inventory not queryable; power connections not tracked; scale-down procedures missing; spot/preemptible not evaluated; I/O reservations missing; GPU monitoring missing |
| INFO | Comprehensive monitoring stack with year-over-year trending; automated capacity reports shared with stakeholders; right-sizing feedback loop validated; provisioning pipeline under 30 minutes from PXE to production-ready; zero configuration drift detected on scheduled CM runs; inventory 100% reconciled with physical audit |

---

## References

- NIST SP 800-123: Guide to General Server Security
- Google SRE Book: Chapter on Capacity Planning (sre.google/sre-book)
- Brendan Gregg: Systems Performance (capacity planning methodology)
- Prometheus documentation: Recording rules, alerting rules
- Grafana documentation: Dashboard best practices
- Red Hat Satellite / Foreman documentation
- Canonical MAAS (Metal as a Service) documentation
- OpenStack Ironic bare-metal provisioning documentation
- Ansible Best Practices documentation (docs.ansible.com)
- Puppet Enterprise documentation (puppet.com)
- Chef Infra documentation (docs.chef.io)
- NetBox documentation (docs.netbox.dev)
- CIS Benchmarks for server operating systems
- Kubernetes HPA documentation (kubernetes.io)
- Linux kernel documentation: cgroups v2, sysctl, tuned
