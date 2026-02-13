# Metrics Instrumentation Reference Checklist

Canonical reference for evaluating metrics instrumentation quality, coverage of
the RED and USE methods, metric type selection, naming conventions, label
cardinality, and business metric presence. Complements the inlined criteria in
`review-observability.md`.

Use this checklist during both diff-based and full-repo reviews. Each section maps
to a category in the finding schema. Severity levels follow
`criteria/shared/severity-levels.md`.

---

## 1. RED Method (Services)

### What to Look For

The RED method defines the minimum metrics every request-handling service must
expose. RED stands for Rate, Errors, Duration. If a service handles requests and
does not expose RED metrics, operators cannot answer the question "is this service
healthy?" without reading logs.

**RED metrics**:

| Metric | What It Measures | Metric Type | Example Name |
|--------|-----------------|-------------|--------------|
| **Rate** | Requests per second | Counter | `http_requests_total` |
| **Errors** | Failed requests per second | Counter | `http_requests_total{status="5xx"}` or `http_request_errors_total` |
| **Duration** | Time to process each request | Histogram | `http_request_duration_seconds` |

**Detection of missing RED metrics**:
```regex
# HTTP handler registration without corresponding metric instrumentation
(?i)(app\.(get|post|put|delete|patch)|router\.(get|post|put|delete)|HandleFunc|@app\.route)
# Check for absence of corresponding metric recording nearby
# Prometheus metric declaration patterns
(?i)(counter|histogram|summary|gauge).*(?:request|http|grpc|rpc)
```

**What a complete RED implementation looks like**:

Each request-handling endpoint should:
1. Increment a request counter on every request (with method, path/handler, status labels)
2. Record request duration in a histogram on every request
3. Errors are captured either as a separate counter or as a label on the request counter (status code >= 500)

### Severity Guidance

| Context | Severity |
|---------|----------|
| No RED metrics on a production request-handling service | HIGH |
| Rate metric present but no duration (latency invisible) | HIGH |
| Rate and duration present but no error differentiation | MEDIUM |
| RED metrics present but only on some endpoints (partial coverage) | MEDIUM |
| RED metrics on all endpoints with consistent labeling | INFO (positive) |

### Fix Patterns

**Python (Flask + Prometheus client)**:
```python
from prometheus_client import Counter, Histogram, generate_latest
from flask import Flask, request, Response
import time

app = Flask(__name__)

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "handler", "status"],
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "handler"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

@app.before_request
def start_timer():
    request._start_time = time.monotonic()

@app.after_request
def record_metrics(response):
    duration = time.monotonic() - request._start_time
    handler = request.endpoint or "unknown"
    REQUEST_COUNT.labels(
        method=request.method,
        handler=handler,
        status=response.status_code,
    ).inc()
    REQUEST_DURATION.labels(
        method=request.method,
        handler=handler,
    ).observe(duration)
    return response

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype="text/plain")
```

**Go (net/http + Prometheus client)**:
```go
package metrics

import (
    "net/http"
    "strconv"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests.",
        },
        []string{"method", "handler", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds.",
            Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0},
        },
        []string{"method", "handler"},
    )
)

// InstrumentHandler wraps an http.Handler with RED metrics.
func InstrumentHandler(handler string, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(rw, r)

        duration := time.Since(start).Seconds()
        httpRequestsTotal.WithLabelValues(r.Method, handler, strconv.Itoa(rw.statusCode)).Inc()
        httpRequestDuration.WithLabelValues(r.Method, handler).Observe(duration)
    })
}

// Expose /metrics endpoint
func MetricsHandler() http.Handler {
    return promhttp.Handler()
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

**Node.js (Express + prom-client)**:
```javascript
const promClient = require('prom-client');

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'handler', 'status'],
});

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'handler'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
});

function metricsMiddleware(req, res, next) {
  const start = process.hrtime.bigint();
  const handler = req.route?.path || req.path || 'unknown';

  res.on('finish', () => {
    const durationNs = Number(process.hrtime.bigint() - start);
    const durationSeconds = durationNs / 1e9;

    httpRequestsTotal.labels(req.method, handler, String(res.statusCode)).inc();
    httpRequestDuration.labels(req.method, handler).observe(durationSeconds);
  });

  next();
}

app.use(metricsMiddleware);
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

**Java (Spring Boot Actuator + Micrometer)**:
```java
// Spring Boot auto-configures RED metrics via Micrometer when actuator is present.
// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    tags:
      application: payment-service
    distribution:
      percentiles-histogram:
        http.server.requests: true
      sla:
        http.server.requests: 50ms,100ms,250ms,500ms,1s,5s

// Custom metrics for non-HTTP endpoints
@Component
public class OrderMetrics {
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;

    public OrderMetrics(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders_total")
            .description("Total orders processed")
            .tag("type", "created")
            .register(registry);
        this.orderProcessingTimer = Timer.builder("order_processing_duration_seconds")
            .description("Order processing duration")
            .publishPercentileHistogram()
            .register(registry);
    }
}
```

---

## 2. USE Method (Resources)

### What to Look For

The USE method defines the minimum metrics for infrastructure resources. USE
stands for Utilization, Saturation, Errors. It applies to resources like CPU,
memory, disk, network interfaces, connection pools, thread pools, and queues.

**USE metrics per resource type**:

| Resource | Utilization | Saturation | Errors |
|----------|------------|------------|--------|
| **CPU** | `process_cpu_seconds_total` | Run queue length, load average | (Typically OS-level) |
| **Memory** | `process_resident_memory_bytes` | Swap usage, OOM kills | Allocation failures |
| **Disk** | `disk_io_utilization` | IO queue depth | IO errors |
| **Network** | `network_transmit_bytes_total` | TX/RX queue drops | Interface errors, retransmits |
| **Connection pool** | `pool_active_connections / pool_max_connections` | `pool_pending_requests` | `pool_connection_errors_total` |
| **Thread pool** | `pool_active_threads / pool_max_threads` | `pool_queued_tasks` | `pool_rejected_tasks_total` |
| **Message queue** | Consumer lag | Queue depth | Dead letter count |
| **File descriptors** | `process_open_fds / process_max_fds` | (At limit = saturated) | "Too many open files" |

**Detection of missing USE metrics**:
```regex
# Connection pool creation without metrics
(?i)(pool|Pool|ConnectionPool|DataSource)\s*\((?!.*(?:metric|gauge|counter|monitor))
# Thread pool creation without metrics
(?i)(ThreadPoolExecutor|ExecutorService|worker_threads)(?!.*(?:metric|gauge|monitor))
# Queue creation without metrics
(?i)(Queue|Channel|queue_declare|createQueue)(?!.*(?:metric|gauge|counter|depth))
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No connection pool metrics (pool exhaustion invisible until outage) | HIGH |
| No thread pool metrics in a service under load | HIGH |
| No memory metrics for a process known to have memory issues | MEDIUM |
| No queue depth metrics in an async-processing service | MEDIUM |
| Missing file descriptor monitoring for a service with many connections | MEDIUM |
| CPU/memory metrics available via infrastructure monitoring but not application-level | LOW |
| USE metrics present for all critical resources with alerting thresholds | INFO (positive) |

### Fix Patterns

**Connection pool metrics (Python + psycopg2 + Prometheus)**:
```python
from prometheus_client import Gauge
import psycopg2.pool

pool = psycopg2.pool.ThreadedConnectionPool(
    minconn=5, maxconn=20,
    dsn="postgresql://localhost/mydb",
)

# Expose pool metrics
POOL_SIZE = Gauge("db_connection_pool_size", "Total connections in pool")
POOL_ACTIVE = Gauge("db_connection_pool_active", "Active connections in pool")
POOL_IDLE = Gauge("db_connection_pool_idle", "Idle connections in pool")

def update_pool_metrics():
    """Call periodically or via middleware."""
    # Implementation depends on pool library
    POOL_SIZE.set(pool.maxconn)
    POOL_ACTIVE.set(pool._used)  # internal attribute; wrap appropriately
    POOL_IDLE.set(pool.maxconn - pool._used)
```

**Thread pool metrics (Java + Micrometer)**:
```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;

@Bean
public ExecutorService taskExecutor(MeterRegistry registry) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(
        10, 50, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000)
    );
    // Wraps executor with Micrometer metrics
    return ExecutorServiceMetrics.monitor(registry, executor, "task_executor");
    // Exposes: executor_pool_size, executor_active, executor_queued,
    //          executor_completed_total, executor_pool_max
}
```

**Queue depth metrics (Go + Prometheus)**:
```go
var (
    queueDepth = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "work_queue_depth",
        Help: "Number of items waiting in the work queue.",
    })
    queueProcessed = promauto.NewCounter(prometheus.CounterOpts{
        Name: "work_queue_processed_total",
        Help: "Total number of items processed from the work queue.",
    })
    queueErrors = promauto.NewCounter(prometheus.CounterOpts{
        Name: "work_queue_errors_total",
        Help: "Total number of processing errors from the work queue.",
    })
)

func enqueue(item WorkItem) {
    queue <- item
    queueDepth.Inc()
}

func processQueue() {
    for item := range queue {
        queueDepth.Dec()
        if err := process(item); err != nil {
            queueErrors.Inc()
            continue
        }
        queueProcessed.Inc()
    }
}
```

---

## 3. Metric Types

### When to Use Each Type

Choosing the wrong metric type produces confusing or incorrect dashboards and
alerts. Each type has a specific purpose.

| Type | Purpose | Behavior | Example |
|------|---------|----------|---------|
| **Counter** | Cumulative count of events | Only goes up (resets on restart) | `http_requests_total`, `errors_total`, `bytes_sent_total` |
| **Gauge** | Current value of something | Goes up and down | `temperature`, `active_connections`, `queue_depth`, `memory_bytes` |
| **Histogram** | Distribution of values in configurable buckets | Records values into pre-defined buckets | `http_request_duration_seconds`, `response_size_bytes` |
| **Summary** | Distribution of values with pre-calculated quantiles | Client-side quantile calculation | `rpc_duration_seconds{quantile="0.99"}` |

**Decision tree**:

```
Is the value cumulative (only increases)?
├── Yes → Counter
│   Example: total requests, total errors, total bytes
└── No, the value can go up and down?
    ├── Yes → Gauge
    │   Example: current temperature, active threads, queue depth
    └── Need to understand the distribution of values?
        ├── Need server-side aggregation across instances? → Histogram
        │   Example: request latency, response sizes
        └── Need pre-calculated quantiles on a single instance? → Summary
            (Note: Summaries cannot be aggregated across instances;
             prefer Histograms for most use cases)
```

**Detection of wrong metric type usage**:
```regex
# Gauge used for cumulative value (should be Counter)
(?i)gauge.*(?:total|count|errors|requests|bytes_sent)
# Counter used for current value (should be Gauge)
(?i)counter.*(?:current|active|depth|size|temperature|utilization)
# Summary where Histogram would be better (multi-instance aggregation needed)
(?i)summary.*(?:http|request|response|latency|duration)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Counter used for a value that decreases (fundamentally broken metric) | HIGH |
| Gauge used for cumulative count (produces incorrect rate() calculations) | HIGH |
| Summary used for latency in a multi-instance service (cannot aggregate p99 across instances) | MEDIUM |
| Histogram with inappropriate bucket boundaries (all values in one bucket) | MEDIUM |
| Minor type mismatch with limited operational impact | LOW |
| Correct metric types used consistently throughout the service | INFO (positive) |

### Common Mistakes

**Mistake**: Using a Gauge for a cumulative count.

Why it is a problem: When you use `rate()` on a Gauge in PromQL, it does not
account for counter resets (process restarts). A restarting process will show
negative rates, producing confusing graphs and incorrect alerts.

Before (smell):
```python
# BAD - Gauge for cumulative value
request_count = Gauge("http_requests", "Total HTTP requests")

@app.after_request
def count_request(response):
    request_count.inc()  # Gauge.inc() works but rate() will break on restart
    return response
```

After (refactored):
```python
# GOOD - Counter for cumulative value
request_count = Counter("http_requests_total", "Total HTTP requests", ["status"])

@app.after_request
def count_request(response):
    request_count.labels(status=response.status_code).inc()
    return response
```

**Mistake**: Using a Summary for latency in a horizontally scaled service.

Why it is a problem: Summary quantiles (p50, p99) are calculated on the client
and cannot be aggregated across instances. If you have 10 instances each
reporting p99=100ms, the actual cluster-wide p99 could be anything from 100ms to
much higher. Histograms allow server-side aggregation using `histogram_quantile()`.

---

## 4. Naming Conventions

### What to Look For

Metric names are the API of your monitoring system. Inconsistent or unclear names
make dashboards and alerts harder to build and maintain.

**Prometheus naming rules**:

| Rule | Example | Anti-Example |
|------|---------|-------------|
| Use `snake_case` | `http_request_duration_seconds` | `httpRequestDuration`, `HTTP-Request-Duration` |
| Include the unit as a suffix | `_seconds`, `_bytes`, `_total` | `_ms`, `_kb`, `_count` (non-standard suffixes) |
| Use base units (seconds not milliseconds, bytes not kilobytes) | `request_duration_seconds` | `request_duration_ms` |
| End counters with `_total` | `http_requests_total` | `http_requests`, `http_request_count` |
| Include a meaningful prefix (namespace) | `myapp_http_requests_total` | `requests_total` |
| Use `_info` suffix for info metrics (Gauge with value 1) | `build_info{version="1.2.3"}` | `version{value="1.2.3"}` |
| Use `_created` suffix for counter creation timestamp | `http_requests_created` | (auto-generated by some clients) |

**Standard metric name patterns**:

| Pattern | Meaning |
|---------|---------|
| `<namespace>_<subsystem>_<name>_<unit>` | Full qualified name |
| `http_requests_total` | Request counter |
| `http_request_duration_seconds` | Request latency histogram |
| `http_request_size_bytes` | Request body size |
| `http_response_size_bytes` | Response body size |
| `process_cpu_seconds_total` | Process CPU usage counter |
| `process_resident_memory_bytes` | Process memory gauge |
| `<resource>_pool_active_connections` | Connection pool utilization |
| `<resource>_pool_max_connections` | Connection pool capacity |

**Detection of naming violations**:
```regex
# camelCase metric names
(?i)(counter|histogram|gauge|summary)\w*\(\s*["'][a-z]+[A-Z]\w*["']
# Missing _total suffix on counters
(?i)counter\w*\(\s*["'][^"']+(?<!_total)["']
# Millisecond units instead of seconds
(?i)(counter|histogram)\w*\(\s*["'][^"']*_m(illi)?s(econds?)?["']
# Kilobyte units instead of bytes
(?i)(counter|histogram)\w*\(\s*["'][^"']*_kb["']
# Missing namespace/prefix (single-word metric name)
(?i)(counter|histogram|gauge)\w*\(\s*["'][a-z_]+["']\s*,
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Counter without `_total` suffix (breaks PromQL conventions and some tooling) | MEDIUM |
| Non-base units (milliseconds, kilobytes) causing dashboard confusion | MEDIUM |
| camelCase or inconsistent naming across metrics | LOW |
| Missing namespace prefix (collision risk in federated setups) | LOW |
| Single-character or meaningless metric names | MEDIUM |
| Consistent naming following Prometheus conventions throughout | INFO (positive) |

---

## 5. Label Cardinality Management

### What to Look For

Labels (also called tags or dimensions) add context to metrics but each unique
combination of label values creates a new time series. High-cardinality labels
(labels with many unique values) can overwhelm the metrics storage backend,
causing memory exhaustion, slow queries, and increased costs.

**High-cardinality anti-patterns**:

| Anti-Pattern | Why It Is Dangerous | Cardinality |
|-------------|--------------------:|-------------|
| User ID as a label | Millions of unique values | O(users) |
| Request ID / Trace ID as a label | Unique per request | O(requests) |
| Full URL path as a label (with IDs) | `/users/123`, `/users/456`... | O(entities) |
| Timestamp as a label | Unique per second | O(seconds) |
| Error message as a label | Unique per error variation | O(error variants) |
| IP address as a label | Up to millions | O(clients) |
| Full SQL query as a label | Unique per query | O(queries) |
| Session ID as a label | Unique per session | O(sessions) |

**Detection of high-cardinality labels**:
```regex
# User/entity IDs in metric labels
(?i)labels?\s*\(.*(?:user_id|customer_id|order_id|request_id|trace_id|session_id|ip_address)
# Full URL paths likely containing IDs
(?i)labels?\(.*(?:req\.url|request\.path|r\.URL\.Path)(?!.*(?:route|handler|endpoint))
# Dynamic error messages as labels
(?i)labels?\(.*(?:err\.message|error\.message|str\(e\)|e\.getMessage)
# Unparameterized paths
(?i)labels?\(.*(?:request\.url|req\.originalUrl|r\.RequestURI)
```

**Safe label values** (bounded cardinality):

| Label | Values | Cardinality |
|-------|--------|-------------|
| `method` | GET, POST, PUT, DELETE, PATCH | 5-7 |
| `handler` / `route` | Named routes (parameterized) | O(endpoints) ~10-100 |
| `status` or `status_code` | 200, 201, 400, 404, 500... | ~20 |
| `service` | Named services | O(services) ~5-50 |
| `environment` | prod, staging, dev | 3-5 |
| `error_type` | Exception class names | O(exception types) ~10-50 |

### Severity Guidance

| Context | Severity |
|---------|----------|
| User ID, session ID, or request ID used as metric label | HIGH |
| Full URL path (with entity IDs) as metric label | HIGH |
| Error message string as metric label | HIGH |
| IP address as metric label in a high-traffic service | MEDIUM |
| Unparameterized URL path (could contain IDs) | MEDIUM |
| Slightly high cardinality label that is bounded (~100-500 values) | LOW |
| All labels have bounded, low cardinality | INFO (positive) |

### Fix Patterns

**Parameterize URL paths before using as labels**:

Before (smell):
```python
# BAD - unbounded cardinality from URL with entity IDs
@app.after_request
def record_metrics(response):
    REQUEST_COUNT.labels(
        method=request.method,
        path=request.path,  # "/users/123", "/users/456", etc.
        status=response.status_code,
    ).inc()
```

After (refactored):
```python
# GOOD - use route template, not resolved path
@app.after_request
def record_metrics(response):
    handler = request.endpoint or "unknown"  # e.g., "get_user"
    # Or use url_rule: request.url_rule.rule -> "/users/<user_id>"
    REQUEST_COUNT.labels(
        method=request.method,
        handler=handler,
        status=response.status_code,
    ).inc()
```

**Categorize error types instead of using error messages**:

Before (smell):
```go
// BAD - error message has unbounded cardinality
errorsTotal.WithLabelValues(err.Error()).Inc()
```

After (refactored):
```go
// GOOD - error type has bounded cardinality
func errorCategory(err error) string {
    switch {
    case errors.Is(err, context.DeadlineExceeded):
        return "timeout"
    case errors.Is(err, sql.ErrNoRows):
        return "not_found"
    case errors.Is(err, sql.ErrConnDone):
        return "connection_error"
    default:
        return "internal"
    }
}

errorsTotal.WithLabelValues(errorCategory(err)).Inc()
```

---

## 6. Histogram Bucket Selection

### What to Look For

Histogram bucket boundaries determine the precision of percentile calculations.
Too few buckets or poorly chosen boundaries produce inaccurate percentiles. The
default Prometheus buckets are designed for HTTP request latency but may be wrong
for other use cases.

**Default Prometheus buckets**:
```
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0
```

These are appropriate for HTTP request latency where most requests complete in
milliseconds to a few seconds. They are NOT appropriate for:

| Use Case | Problem with Defaults | Better Buckets |
|----------|----------------------|----------------|
| Sub-millisecond operations (cache lookups) | No resolution below 5ms | `0.0001, 0.0005, 0.001, 0.005, 0.01` |
| Long-running operations (batch jobs, reports) | Max bucket is 10s | `1, 5, 10, 30, 60, 120, 300, 600` |
| Message processing latency | May need sub-ms to seconds range | `0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0` |
| Response body sizes (bytes) | Latency buckets are meaningless for sizes | `100, 1000, 10000, 100000, 1000000` |

**Detection of bucket issues**:
```regex
# Default buckets used for non-HTTP metrics
(?i)histogram\w*\((?!.*buckets).*(?:batch|queue|job|report|file|download|upload)
# Very few custom buckets (poor resolution)
(?i)buckets\s*[:=]\s*\[?\s*\d+\.?\d*\s*,\s*\d+\.?\d*\s*\]
# Buckets that do not match the metric's expected range
# (requires understanding the metric's domain)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Default buckets used for non-HTTP-latency metrics (e.g., batch job duration) | MEDIUM |
| Only 2-3 bucket boundaries (very poor percentile resolution) | MEDIUM |
| Bucket boundaries that miss the expected range entirely | HIGH |
| No custom buckets considered for domain-specific metrics | LOW |
| Well-chosen buckets matching the metric's expected value distribution | INFO (positive) |

### Fix Pattern

**Choose buckets based on SLO targets**:

If your SLO states "99% of requests complete in under 500ms", your buckets need
resolution around the 500ms boundary to accurately measure this:

```python
# Buckets chosen to give resolution around the 500ms SLO target
REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["handler"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.4, 0.5, 0.75, 1.0, 2.5, 5.0, 10.0],
    #                                          ^^^^  ^^^  ^^^  ^^^
    #                              Dense buckets around the SLO boundary
)
```

**Exponential bucket generation**:
```python
# For use cases where the range spans several orders of magnitude
from prometheus_client import Histogram

def exponential_buckets(start, factor, count):
    return [start * (factor ** i) for i in range(count)]

PROCESSING_DURATION = Histogram(
    "batch_processing_duration_seconds",
    "Batch processing duration",
    buckets=exponential_buckets(0.1, 2, 12),
    # Produces: 0.1, 0.2, 0.4, 0.8, 1.6, 3.2, 6.4, 12.8, 25.6, 51.2, 102.4, 204.8
)
```

---

## 7. Metric Collection

### What to Look For

Metrics must be collected reliably and efficiently. The collection architecture
affects both the accuracy of the data and the operational overhead.

**Collection requirements**:

| Requirement | Description |
|-------------|-------------|
| **`/metrics` endpoint** | Prometheus-compatible endpoint for pull-based collection |
| **Collection interval** | Scrape interval aligned with metric granularity (typically 15-30s) |
| **Metric initialization** | Metrics initialized at startup (not lazily on first request) |
| **Cardinality bounds** | Total time series per instance stays within backend limits |
| **Metric registry** | Single registry per process (avoid duplicate registrations) |
| **Metric documentation** | Each metric has a help string describing its purpose |

**Detection of collection issues**:
```regex
# Missing /metrics endpoint
# (Check for absence of metrics handler registration)
(?i)(promhttp\.Handler|generate_latest|register\.metrics|prometheus.*endpoint)
# Lazy metric initialization (metrics not initialized at module level)
(?i)def\s+\w+\(.*\):[^}]*(Counter|Histogram|Gauge)\s*\(
# Missing help string
(?i)(Counter|Histogram|Gauge|Summary)\s*\(\s*["'][^"']+["']\s*,\s*["']\s*["']
# Duplicate metric registration errors
(?i)(already.*registered|duplicate.*metric|AlreadyRegisteredError)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No `/metrics` endpoint in a service that should expose metrics | HIGH |
| Metrics created inside request handlers (new metric per request = memory leak) | HIGH |
| No metric initialization at startup (first-request latency spike; missing zero values) | MEDIUM |
| Missing help strings on metric definitions | LOW |
| Scrape interval not configured or using defaults without consideration | LOW |
| Properly initialized metrics with `/metrics` endpoint and documentation | INFO (positive) |

### Common Mistakes

**Mistake**: Creating metrics inside request handlers.

Why it is a problem: If a Counter or Histogram is created inside a function that
runs per-request, each invocation creates a new metric object. In some client
libraries this causes registration errors; in others it causes a memory leak.

Before (smell):
```python
@app.route("/api/orders")
def get_orders():
    # BAD - creates a new Counter on every request
    counter = Counter("order_requests_total", "Order requests")
    counter.inc()
    return get_all_orders()
```

After (refactored):
```python
# GOOD - metric defined at module level, initialized once
ORDER_REQUESTS = Counter("order_requests_total", "Total order list requests")

@app.route("/api/orders")
def get_orders():
    ORDER_REQUESTS.inc()
    return get_all_orders()
```

**Mistake**: Not initializing metric label combinations at startup.

Why it is a problem: Prometheus only reports time series that have been observed.
If a label combination has never been incremented, it will not appear in queries.
This means `rate(http_requests_total{status="500"})` returns no data instead of 0,
making alerts unreliable.

```python
# Initialize all expected label combinations at startup
REQUEST_COUNT = Counter("http_requests_total", "Total requests", ["method", "status"])

# Pre-initialize known combinations so they appear as 0 from the start
for method in ["GET", "POST", "PUT", "DELETE"]:
    for status in ["200", "400", "404", "500"]:
        REQUEST_COUNT.labels(method=method, status=status)
```

---

## 8. Business Metrics

### What to Look For

Technical metrics (RED, USE) tell you if the system is healthy. Business metrics
tell you if the system is achieving its purpose. Without business metrics,
operators can see that the server is up and fast but cannot tell if customers are
actually placing orders, payments are processing, or the core function is working.

**Business metric examples**:

| Domain | Metric | Type | Labels |
|--------|--------|------|--------|
| E-commerce | `orders_placed_total` | Counter | payment_method, currency |
| E-commerce | `order_value_dollars` | Histogram | payment_method |
| E-commerce | `cart_abandonment_total` | Counter | step (shipping, payment, review) |
| Auth | `login_attempts_total` | Counter | result (success, failure, locked) |
| Auth | `active_sessions` | Gauge | |
| Messaging | `messages_sent_total` | Counter | channel (email, sms, push) |
| Messaging | `message_delivery_duration_seconds` | Histogram | channel |
| Payments | `payment_processed_total` | Counter | provider, status |
| Payments | `payment_amount_dollars` | Histogram | provider, currency |
| SaaS | `api_calls_by_tenant_total` | Counter | plan_tier |
| SaaS | `feature_usage_total` | Counter | feature_name |

**Detection of missing business metrics**:
```regex
# Business operations without corresponding metric recording
(?i)def\s+(create_order|place_order|process_payment|send_message|register_user|login)\w*\(
# Check for absence of Counter/Histogram recording in business-critical functions
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No business metrics in a revenue-generating service | HIGH |
| Business metrics present but incomplete (e.g., orders counted but not revenue) | MEDIUM |
| Business metrics present but not broken down by meaningful dimensions | LOW |
| Technical metrics only; no business visibility | MEDIUM |
| Comprehensive business metrics alongside RED/USE | INFO (positive) |

---

## 9. Prometheus Client Library Patterns

### Python (prometheus_client)

```python
from prometheus_client import (
    Counter, Histogram, Gauge, Summary, Info,
    generate_latest, CONTENT_TYPE_LATEST,
    CollectorRegistry, REGISTRY,
)

# Standard metric declarations (module level)
REQUESTS = Counter(
    "myapp_http_requests_total",
    "Total HTTP requests processed.",
    ["method", "handler", "status"],
    registry=REGISTRY,
)

LATENCY = Histogram(
    "myapp_http_request_duration_seconds",
    "HTTP request duration in seconds.",
    ["method", "handler"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

IN_PROGRESS = Gauge(
    "myapp_http_requests_in_progress",
    "Number of HTTP requests currently being processed.",
    ["handler"],
)

BUILD_INFO = Info(
    "myapp_build",
    "Build information.",
)
BUILD_INFO.info({
    "version": "1.2.3",
    "commit": "abc123",
    "build_date": "2025-03-15",
})
```

### Go (prometheus/client_golang)

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    Requests = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "myapp",
            Subsystem: "http",
            Name:      "requests_total",
            Help:      "Total HTTP requests processed.",
        },
        []string{"method", "handler", "status"},
    )

    Latency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "myapp",
            Subsystem: "http",
            Name:      "request_duration_seconds",
            Help:      "HTTP request duration in seconds.",
            Buckets:   []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0},
        },
        []string{"method", "handler"},
    )

    InProgress = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Namespace: "myapp",
            Subsystem: "http",
            Name:      "requests_in_progress",
            Help:      "Number of HTTP requests currently being processed.",
        },
        []string{"handler"},
    )

    BuildInfo = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Namespace: "myapp",
            Name:      "build_info",
            Help:      "Build information.",
        },
        []string{"version", "commit", "build_date"},
    )
)

func init() {
    BuildInfo.WithLabelValues("1.2.3", "abc123", "2025-03-15").Set(1)
}
```

### Node.js (prom-client)

```javascript
const promClient = require('prom-client');

// Collect default process metrics (CPU, memory, event loop, etc.)
promClient.collectDefaultMetrics({
  prefix: 'myapp_',
  labels: { service: 'myapp' },
});

const requests = new promClient.Counter({
  name: 'myapp_http_requests_total',
  help: 'Total HTTP requests processed.',
  labelNames: ['method', 'handler', 'status'],
});

const latency = new promClient.Histogram({
  name: 'myapp_http_request_duration_seconds',
  help: 'HTTP request duration in seconds.',
  labelNames: ['method', 'handler'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
});

const inProgress = new promClient.Gauge({
  name: 'myapp_http_requests_in_progress',
  help: 'Number of HTTP requests currently being processed.',
  labelNames: ['handler'],
});

module.exports = { requests, latency, inProgress };
```

### Java (Micrometer)

```java
import io.micrometer.core.instrument.*;

@Component
public class AppMetrics {
    private final Counter requestsTotal;
    private final Timer requestDuration;
    private final AtomicInteger inProgress;

    public AppMetrics(MeterRegistry registry) {
        this.requestsTotal = Counter.builder("myapp.http.requests.total")
            .description("Total HTTP requests processed.")
            .tag("service", "myapp")
            .register(registry);

        this.requestDuration = Timer.builder("myapp.http.request.duration")
            .description("HTTP request duration.")
            .publishPercentileHistogram()
            .serviceLevelObjectives(
                Duration.ofMillis(50),
                Duration.ofMillis(100),
                Duration.ofMillis(250),
                Duration.ofMillis(500),
                Duration.ofSeconds(1)
            )
            .register(registry);

        // Gauge for in-progress requests
        this.inProgress = registry.gauge(
            "myapp.http.requests.in_progress",
            new AtomicInteger(0)
        );

        // Build info
        Gauge.builder("myapp.build.info", () -> 1)
            .tag("version", "1.2.3")
            .tag("commit", "abc123")
            .description("Build information.")
            .register(registry);
    }
}
```

---

## Review Procedure Summary

When evaluating metrics instrumentation:

1. **Check RED metrics**: Verify every request-handling service exposes Rate (counter), Errors (counter or label), and Duration (histogram) metrics for all endpoints.
2. **Check USE metrics**: Verify resource metrics (connection pools, thread pools, queues, memory) are instrumented for all critical infrastructure components the service manages.
3. **Verify metric types**: Confirm Counters are used for cumulative values, Gauges for current values, and Histograms for distributions. Flag type misuse.
4. **Audit naming conventions**: Check for snake_case, base units (seconds, bytes), `_total` suffix on counters, and meaningful namespace prefixes.
5. **Check label cardinality**: Flag any labels that could have unbounded values (user IDs, request IDs, full URLs, error messages, IP addresses).
6. **Evaluate histogram buckets**: Verify bucket boundaries match the expected value range and provide resolution around SLO thresholds.
7. **Verify collection infrastructure**: Confirm `/metrics` endpoint exists, metrics are initialized at startup, and no metrics are created inside request handlers.
8. **Check business metrics**: Verify business-relevant counters and histograms exist alongside technical metrics.
9. **Classify each finding** using the severity tables above, adjusting for the service's criticality and traffic volume.

---

## Quick Reference: Severity Summary

| Severity | Metrics Findings |
|----------|-----------------|
| CRITICAL | (Metrics issues are not typically CRITICAL alone; they become CRITICAL when combined with operational blindness during an active incident) |
| HIGH | No RED metrics on a request-handling service; no duration metric (latency invisible); counter used for decreasing value or gauge for cumulative count; user/request IDs as metric labels; metrics created inside request handlers; no /metrics endpoint; no connection pool metrics; no business metrics in revenue-generating service; histogram buckets completely miss expected range |
| MEDIUM | Partial RED coverage (some endpoints missing); no error differentiation in metrics; non-standard naming (wrong units, missing _total); default histogram buckets for non-HTTP use case; no metric initialization at startup; summary used where histogram needed; unparameterized URL paths in labels; queue depth not instrumented |
| LOW | Minor naming inconsistencies; missing namespace prefix; missing help strings; scrape interval not tuned; business metric dimensions too coarse; slightly high cardinality label that is bounded |
| INFO | Complete RED and USE coverage; correct metric types; consistent naming; bounded label cardinality; well-chosen histogram buckets; comprehensive business metrics; proper initialization |

---

## References

- [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/) -- Counter, Gauge, Histogram, Summary definitions
- [Prometheus Naming Best Practices](https://prometheus.io/docs/practices/naming/) -- official naming conventions
- [RED Method (Tom Wilkie)](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/) -- Rate, Errors, Duration for services
- [USE Method (Brendan Gregg)](https://www.brendangregg.com/usemethod.html) -- Utilization, Saturation, Errors for resources
- [Google SRE Book, Chapter 6: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) -- the four golden signals
- [OpenTelemetry Metrics Specification](https://opentelemetry.io/docs/specs/otel/metrics/) -- standard metrics data model
- [Prometheus Histograms and Summaries](https://prometheus.io/docs/practices/histograms/) -- when to use each
- [Prometheus Client Libraries](https://prometheus.io/docs/instrumenting/clientlibs/) -- official client library list
