# Log Correlation Reference Checklist

Canonical reference for evaluating request correlation, cross-service context
propagation, log aggregation architecture, and correlation strategies across
different system architectures. Complements the inlined criteria in
`review-observability.md`.

Use this checklist during both diff-based and full-repo reviews. Each section maps
to a category in the finding schema. Severity levels follow
`criteria/shared/severity-levels.md`.

---

## 1. Request Correlation

### What to Look For

Request correlation assigns a unique identifier to every incoming request and
ensures that identifier appears in every log entry produced while handling that
request. Without correlation IDs, diagnosing a failure in a system processing
thousands of concurrent requests is like finding a needle in a haystack of
identical-looking needles.

**Correlation ID requirements**:

| Requirement | Description | Detection |
|-------------|-------------|-----------|
| **Generation** | A unique ID is generated for each incoming request if not already present | Check for UUID/ULID generation in middleware/interceptors |
| **Propagation** | The ID is passed to all downstream calls (HTTP, gRPC, message queues) | Check for header injection in HTTP clients |
| **Inclusion** | The ID appears in every log entry for the request lifecycle | Check for MDC/context variable binding in middleware |
| **Response headers** | The ID is returned in the response for client-side correlation | Check for response header injection |
| **Format** | IDs are globally unique and URL-safe | UUID v4, ULID, or custom format with sufficient entropy |

**Detection of missing correlation**:
```regex
# HTTP handler without request ID extraction or generation
(?i)def\s+(get|post|put|delete|patch|handle)\w*\(.*\):[^}]*(?!request_id|correlation_id|req_id|x.request.id)
# HTTP client calls without forwarding correlation headers
(?i)(requests\.(get|post|put)|http\.Get|fetch\(|axios\.)(?!.*(?:x-request-id|x-correlation-id|traceparent))
# Middleware stack without correlation ID middleware
# (check middleware registration order for missing correlation layer)
```

**Standard header names** (check for consistency):

| Header | Standard | Usage |
|--------|----------|-------|
| `X-Request-ID` | De facto standard | Request correlation within a service boundary |
| `X-Correlation-ID` | De facto standard | End-to-end correlation across service boundaries |
| `traceparent` | W3C Trace Context | Distributed tracing (includes trace_id and span_id) |
| `X-B3-TraceId` | Zipkin B3 | Legacy distributed tracing header |

### Severity Guidance

| Context | Severity |
|---------|----------|
| No request correlation in a multi-service architecture | HIGH |
| Correlation ID generated but not propagated to downstream services | HIGH |
| Correlation ID not included in log entries | MEDIUM |
| Correlation ID not returned in response headers | LOW |
| Inconsistent header names across services (some use X-Request-ID, others X-Correlation-ID) | MEDIUM |
| Multiple correlation schemes without mapping between them | MEDIUM |
| Complete correlation chain from ingress to all downstream services | INFO (positive) |

### Common Mistakes

**Mistake**: Generating a new correlation ID for every inter-service call instead of
propagating the original.

Why it is a problem: If Service A receives request `req_abc`, calls Service B,
and Service B generates its own `req_xyz`, there is no way to connect the logs
from A and B. The whole point of correlation is the shared identifier.

Before (smell):
```python
# Service A - HTTP handler
@app.route("/orders", methods=["POST"])
def create_order():
    # Generates an ID but does not propagate it
    request_id = str(uuid.uuid4())
    logger.info("order_requested", request_id=request_id)

    # Calls Service B without forwarding the ID
    response = requests.post("http://inventory-service/reserve", json=payload)
    return jsonify(result)
```

After (refactored):
```python
# Middleware: extract or generate correlation ID
@app.before_request
def set_correlation_id():
    request_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    g.request_id = request_id
    structlog.contextvars.bind_contextvars(request_id=request_id)

@app.after_request
def add_correlation_header(response):
    response.headers["X-Request-ID"] = g.request_id
    return response

# HTTP client wrapper that propagates correlation
def service_call(method, url, **kwargs):
    headers = kwargs.pop("headers", {})
    headers["X-Request-ID"] = g.request_id
    return requests.request(method, url, headers=headers, **kwargs)

@app.route("/orders", methods=["POST"])
def create_order():
    logger.info("order_requested")  # request_id auto-included via contextvars
    response = service_call("POST", "http://inventory-service/reserve", json=payload)
    return jsonify(result)
```

**Mistake**: Using sequential or predictable correlation IDs.

Why it is a problem: Sequential IDs (auto-increment counters) can collide across
service instances and leak information about request volume. Use UUIDs, ULIDs, or
sufficiently random strings.

Before (smell):
```python
# BAD - counter-based IDs collide across instances
_counter = 0
def get_request_id():
    global _counter
    _counter += 1
    return f"req_{_counter}"
```

After (refactored):
```python
import uuid

def get_request_id():
    return f"req_{uuid.uuid4()}"

# Or use ULID for sortable, unique IDs
import ulid
def get_request_id():
    return f"req_{ulid.new()}"
```

### Framework-Specific Middleware Examples

**Python Flask**:
```python
import uuid
import structlog
from flask import Flask, request, g

app = Flask(__name__)

@app.before_request
def inject_correlation_id():
    request_id = (
        request.headers.get("X-Request-ID")
        or request.headers.get("X-Correlation-ID")
        or str(uuid.uuid4())
    )
    g.request_id = request_id
    # Bind to structlog contextvars for automatic inclusion
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        method=request.method,
        path=request.path,
    )

@app.after_request
def add_response_headers(response):
    response.headers["X-Request-ID"] = g.request_id
    return response
```

**Python FastAPI**:
```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

class CorrelationMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = (
            request.headers.get("X-Request-ID")
            or request.headers.get("X-Correlation-ID")
            or str(uuid.uuid4())
        )
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request_id,
            method=request.method,
            path=request.url.path,
        )
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response

app.add_middleware(CorrelationMiddleware)
```

**Go net/http middleware**:
```go
package middleware

import (
    "context"
    "log/slog"
    "net/http"

    "github.com/google/uuid"
)

type contextKey string

const RequestIDKey contextKey = "request_id"

func CorrelationID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Set on response headers
        w.Header().Set("X-Request-ID", requestID)

        // Set on context for downstream access
        ctx := context.WithValue(r.Context(), RequestIDKey, requestID)

        // Create request-scoped logger
        logger := slog.Default().With(
            slog.String("request_id", requestID),
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
        )
        ctx = context.WithValue(ctx, "logger", logger)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// RequestID extracts the request ID from context.
func RequestID(ctx context.Context) string {
    if id, ok := ctx.Value(RequestIDKey).(string); ok {
        return id
    }
    return ""
}
```

**Node.js Express middleware**:
```javascript
const { randomUUID } = require('crypto');
const { AsyncLocalStorage } = require('async_hooks');

const correlationStore = new AsyncLocalStorage();

function correlationMiddleware(req, res, next) {
  const requestId = req.headers['x-request-id'] || randomUUID();
  res.setHeader('X-Request-ID', requestId);

  const context = {
    requestId,
    method: req.method,
    path: req.path,
  };

  correlationStore.run(context, () => next());
}

function getCorrelationContext() {
  return correlationStore.getStore() || {};
}

// Logger integration
const pino = require('pino');
const logger = pino({
  mixin() {
    return getCorrelationContext();
  },
});

module.exports = { correlationMiddleware, getCorrelationContext, logger };
```

**Java Spring Boot filter**:
```java
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

@Component
public class CorrelationFilter extends OncePerRequestFilter {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws Exception {
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }

        MDC.put("request_id", requestId);
        response.setHeader(REQUEST_ID_HEADER, requestId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("request_id");
        }
    }
}
```

---

## 2. Cross-Service Correlation

### What to Look For

In distributed systems, a single user action can trigger calls across many
services. Cross-service correlation ensures that logs from every service involved
in handling a request can be connected using a shared identifier. This is the
foundation of distributed tracing.

**Context propagation mechanisms**:

| Mechanism | Scope | How It Works |
|-----------|-------|-------------|
| **HTTP headers** | Synchronous calls | Inject correlation headers in outgoing requests; extract from incoming requests |
| **gRPC metadata** | Synchronous calls | Propagate via gRPC metadata (equivalent to HTTP headers) |
| **Message headers** | Async messaging | Include correlation ID in message metadata/headers when publishing |
| **W3C Trace Context** | Cross-vendor | `traceparent` header with trace_id, span_id, trace_flags |
| **B3 propagation** | Zipkin ecosystem | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` headers |
| **Baggage** | Cross-service metadata | `baggage` header for arbitrary key-value propagation |

### W3C Trace Context

The W3C Trace Context is the standard for distributed context propagation. It uses
two headers:

**`traceparent` header format**:
```
traceparent: {version}-{trace-id}-{parent-id}-{trace-flags}
             00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

| Field | Size | Description |
|-------|------|-------------|
| `version` | 2 hex chars | Always `00` in current spec |
| `trace-id` | 32 hex chars | Globally unique trace identifier |
| `parent-id` | 16 hex chars | ID of the parent span |
| `trace-flags` | 2 hex chars | `01` = sampled, `00` = not sampled |

**`tracestate` header** (optional):
```
tracestate: vendor1=value1,vendor2=value2
```

Used for vendor-specific trace information. Multiple vendors can coexist.

**Detection of W3C Trace Context support**:
```regex
# OpenTelemetry propagator configuration
(?i)(TraceContextPropagator|W3CTraceContextPropagator|traceparent|tracecontext)
# Header extraction
(?i)headers?\.(get|extract)\s*\(\s*["']traceparent["']
```

### B3 Propagation (Zipkin)

B3 uses multiple headers (multi-header format) or a single header:

**Multi-header format**:
```
X-B3-TraceId: 463ac35c9f6413ad48485a3953bb6124
X-B3-SpanId: 0020000000000001
X-B3-ParentSpanId: 0000000000000000
X-B3-Sampled: 1
```

**Single-header format**:
```
b3: 463ac35c9f6413ad48485a3953bb6124-0020000000000001-1-0000000000000000
```

**Detection of B3 support**:
```regex
(?i)(B3Propagator|X-B3-TraceId|b3-propagation|zipkin)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No context propagation between services in a distributed system | HIGH |
| Correlation ID in logs but not propagated via HTTP headers to downstream services | HIGH |
| Mixed propagation standards without interop (some W3C, some B3, no bridge) | MEDIUM |
| Context propagated via HTTP but not via message queue headers | MEDIUM |
| Custom propagation format instead of W3C Trace Context or B3 | LOW |
| Missing `tracestate` header (only `traceparent`) | INFO |
| Full W3C Trace Context propagation across all service boundaries | INFO (positive) |

### Common Mistakes

**Mistake**: Propagating correlation via HTTP headers but not via message queues.

Why it is a problem: When Service A publishes a message to a queue and Service B
consumes it, the correlation chain breaks. The logs from the consumer cannot be
connected to the logs from the producer.

Before (smell):
```python
# Producer - publishes message without correlation headers
def publish_order_event(order):
    channel.basic_publish(
        exchange="orders",
        routing_key="order.created",
        body=json.dumps({"order_id": order.id, "amount": order.amount}),
    )
```

After (refactored):
```python
# Producer - includes correlation context in message headers
def publish_order_event(order):
    correlation_id = structlog.contextvars.get_contextvars().get("request_id", str(uuid.uuid4()))
    trace_id = structlog.contextvars.get_contextvars().get("trace_id", "")

    properties = pika.BasicProperties(
        headers={
            "X-Request-ID": correlation_id,
            "X-Trace-ID": trace_id,
            "X-Source-Service": "order-service",
        },
        correlation_id=correlation_id,
        content_type="application/json",
    )
    channel.basic_publish(
        exchange="orders",
        routing_key="order.created",
        body=json.dumps({"order_id": order.id, "amount": order.amount}),
        properties=properties,
    )

# Consumer - extracts correlation context from message headers
def on_order_created(channel, method, properties, body):
    request_id = properties.headers.get("X-Request-ID", str(uuid.uuid4()))
    trace_id = properties.headers.get("X-Trace-ID", "")

    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        trace_id=trace_id,
        source_service=properties.headers.get("X-Source-Service", "unknown"),
    )
    logger.info("order_event_received", order_data=json.loads(body))
```

**Mistake**: Not propagating context through async boundaries (goroutines, threads, async/await).

Before (smell):
```go
// BAD - goroutine loses context
func handleRequest(w http.ResponseWriter, r *http.Request) {
    requestID := r.Header.Get("X-Request-ID")
    logger.Info("request_received", slog.String("request_id", requestID))

    go func() {
        // This goroutine has no access to requestID
        processInBackground()
    }()
}
```

After (refactored):
```go
// GOOD - context passed to goroutine
func handleRequest(w http.ResponseWriter, r *http.Request) {
    requestID := r.Header.Get("X-Request-ID")
    ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
    logger := slog.Default().With(slog.String("request_id", requestID))

    logger.Info("request_received")

    go func(ctx context.Context) {
        reqID := ctx.Value(RequestIDKey).(string)
        bgLogger := slog.Default().With(slog.String("request_id", reqID))
        bgLogger.Info("background_processing_started")
        processInBackground(ctx)
    }(ctx)
}
```

### HTTP Client Wrappers for Propagation

**Python requests wrapper**:
```python
import requests
from flask import g

class CorrelatedSession(requests.Session):
    """HTTP session that automatically propagates correlation headers."""

    def request(self, method, url, **kwargs):
        headers = kwargs.pop("headers", {})
        # Propagate correlation context
        if hasattr(g, "request_id"):
            headers.setdefault("X-Request-ID", g.request_id)
        if hasattr(g, "trace_id"):
            headers.setdefault("traceparent", g.traceparent)
        kwargs["headers"] = headers
        return super().request(method, url, **kwargs)

# Usage
http = CorrelatedSession()
response = http.get("http://inventory-service/stock")  # Headers auto-propagated
```

**Go HTTP client with context propagation**:
```go
// PropagatingTransport injects correlation headers into outgoing requests.
type PropagatingTransport struct {
    Base http.RoundTripper
}

func (t *PropagatingTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    if requestID := RequestID(req.Context()); requestID != "" {
        req.Header.Set("X-Request-ID", requestID)
    }
    // Propagate W3C Trace Context if present
    if traceparent := req.Context().Value("traceparent"); traceparent != nil {
        req.Header.Set("traceparent", traceparent.(string))
    }
    base := t.Base
    if base == nil {
        base = http.DefaultTransport
    }
    return base.RoundTrip(req)
}

// Usage
client := &http.Client{
    Transport: &PropagatingTransport{},
}
```

**Node.js axios interceptor**:
```javascript
const axios = require('axios');
const { getCorrelationContext } = require('./correlation');

const httpClient = axios.create();

httpClient.interceptors.request.use((config) => {
  const ctx = getCorrelationContext();
  if (ctx.requestId) {
    config.headers['X-Request-ID'] = ctx.requestId;
  }
  if (ctx.traceparent) {
    config.headers['traceparent'] = ctx.traceparent;
  }
  return config;
});

module.exports = httpClient;
```

**Java Spring RestTemplate interceptor**:
```java
import org.slf4j.MDC;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;

public class CorrelationInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                         ClientHttpRequestExecution execution) throws Exception {
        String requestId = MDC.get("request_id");
        if (requestId != null) {
            request.getHeaders().set("X-Request-ID", requestId);
        }
        String traceparent = MDC.get("traceparent");
        if (traceparent != null) {
            request.getHeaders().set("traceparent", traceparent);
        }
        return execution.execute(request, body);
    }
}

// Registration
@Bean
public RestTemplate restTemplate() {
    RestTemplate template = new RestTemplate();
    template.setInterceptors(List.of(new CorrelationInterceptor()));
    return template;
}
```

---

## 3. Log Aggregation

### What to Look For

Logs are only useful if they can be searched, correlated, and analyzed in a
centralized location. Log aggregation collects logs from all services and
infrastructure into a single queryable system.

**Aggregation requirements**:

| Requirement | Description |
|-------------|-------------|
| **Centralized shipping** | All services ship logs to a central system (ELK, Loki, Datadog, Splunk, CloudWatch) |
| **Reliability** | Log shipping does not block or crash the application if the aggregation system is unavailable |
| **Format compatibility** | All services emit logs in a format the aggregation system can parse (JSON is the most portable) |
| **Retention policy** | Retention periods are defined and aligned with compliance requirements |
| **Timestamp standardization** | All services use the same timestamp format and timezone (ISO 8601 UTC) |
| **Index/label strategy** | Logs are indexed or labeled by service, environment, and severity for efficient querying |

**Detection of aggregation problems**:
```regex
# File-only logging with no shipping agent
(?i)(?:FileHandler|file_handler|filename|RotatingFileHandler)(?!.*(?:fluentd|filebeat|vector|logstash|promtail))
# Local timezone in timestamps
(?i)(localtime|strftime.*%z|timezone.*local|TimeZone.*(?!UTC))
# Missing log shipping configuration
# (Check for absence of fluentd, filebeat, vector, promtail, or cloud logging agent configs)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No centralized log aggregation in a multi-service system | HIGH |
| Log shipping that blocks the application on aggregation system failure | HIGH |
| Inconsistent timestamp formats across services (some local, some UTC) | MEDIUM |
| Missing retention policy (logs grow unbounded or are deleted too quickly) | MEDIUM |
| No structured format (free-text logs shipped to aggregation) | MEDIUM |
| Different services use different log aggregation backends | MEDIUM |
| Aggregation configured but no alerting on log shipping failures | LOW |
| Missing log shipping health monitoring | LOW |
| Centralized aggregation with consistent format, retention, and alerting | INFO (positive) |

### Timestamp Standardization

**Standard**: ISO 8601 with UTC timezone and millisecond precision.

```
2025-03-15T14:22:33.456Z
```

**Why UTC**: Services running across multiple timezones produce logs that are
impossible to correlate by time if each uses local time. UTC eliminates ambiguity.

**Why ISO 8601**: Human-readable, lexicographically sortable, and universally
supported by log aggregation systems.

**Detection of non-standard timestamps**:
```regex
# Unix epoch timestamps (not human-readable)
"timestamp"\s*:\s*\d{10,13}[,}]
# Non-UTC timezone markers
\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}[+-]\d{2}:?\d{2}
# Non-ISO formats
\d{2}/\d{2}/\d{4}\s+\d{2}:\d{2}:\d{2}
# Missing timezone indicator
\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d+[^Z+-]
```

| Timestamp Format | Assessment | Severity |
|-----------------|------------|----------|
| `2025-03-15T14:22:33.456Z` | Correct: ISO 8601 UTC | INFO (positive) |
| `2025-03-15T14:22:33.456+05:30` | Non-UTC but includes offset (convertible) | LOW |
| `1710512553456` | Epoch milliseconds (not human-readable) | LOW |
| `03/15/2025 14:22:33` | Non-ISO, ambiguous (US vs. EU date), no timezone | MEDIUM |
| `2025-03-15 14:22:33` | Missing timezone, missing T separator | MEDIUM |
| No timestamp in log output | Relies on collection time (inaccurate) | HIGH |

### Log Shipping Patterns

**Docker / Kubernetes: stdout-based shipping**:
```yaml
# Application logs to stdout; infrastructure handles shipping
# Kubernetes: Use fluentd/fluent-bit DaemonSet or Promtail

# fluent-bit configuration (DaemonSet)
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        Refresh_Interval  10
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On

    [FILTER]
        Name              kubernetes
        Match             kube.*
        Kube_URL          https://kubernetes.default.svc:443
        Kube_Tag_Prefix   kube.var.log.containers.
        Merge_Log         On
        Merge_Log_Key     log_processed

    [OUTPUT]
        Name              es
        Match             *
        Host              elasticsearch.logging.svc
        Port              9200
        Logstash_Format   On
        Retry_Limit       False
```

**Non-blocking log shipping (application-level)**:
```python
# Python: Async log handler that does not block the application
import logging
from logging.handlers import QueueHandler, QueueListener
from queue import Queue

def configure_async_shipping(remote_handler):
    """Wrap a remote log handler with async queuing to prevent blocking."""
    log_queue = Queue(maxsize=10000)
    queue_handler = QueueHandler(log_queue)

    listener = QueueListener(log_queue, remote_handler, respect_handler_level=True)
    listener.start()

    root = logging.getLogger()
    root.addHandler(queue_handler)
    return listener  # Call listener.stop() on shutdown
```

---

## 4. Correlation in Different Architectures

### Synchronous Request/Response

The simplest correlation pattern. A request enters the system, is processed
(potentially across multiple services), and a response is returned.

**Correlation chain**:
```
Client -> API Gateway -> Service A -> Service B -> Database
         [generates]    [receives/    [receives/
          request_id     propagates]   propagates]
```

**Requirements**:

| Checkpoint | What to Verify |
|------------|---------------|
| Ingress (API gateway / load balancer) | Generates request_id if not present; passes to backend |
| Service middleware | Extracts request_id from incoming header; binds to log context |
| Outgoing HTTP calls | Includes request_id in headers to downstream services |
| Response | Returns request_id in response headers to caller |
| Database queries | Request_id available for slow query correlation |

**Severity guidance for synchronous patterns**:

| Context | Severity |
|---------|----------|
| API gateway does not generate or propagate correlation IDs | HIGH |
| Service receives correlation ID but does not include it in downstream calls | HIGH |
| Correlation ID present in application logs but not in access logs (nginx, ALB) | MEDIUM |
| Database query logs not correlated with application request_id | LOW |
| Full correlation from ingress through all services to response | INFO (positive) |

### Asynchronous Message Queues

Correlation across message queues requires explicit header propagation because
there is no HTTP request/response to carry the context automatically.

**Correlation chain**:
```
Service A -> Message Queue -> Service B -> Service C
[publishes    [msg headers    [consumes,    [receives
 with          carry           extracts      propagated
 headers]      context]        context]      context]
```

**Requirements**:

| Checkpoint | What to Verify |
|------------|---------------|
| Message producer | Injects correlation_id, trace_id, source_service into message headers |
| Message consumer | Extracts correlation context from message headers before processing |
| Consumer log context | Binds extracted IDs to the log context for the processing scope |
| Dead letter queue | Correlation context preserved when messages move to DLQ |
| Reply messages | If a reply is sent, the original correlation_id is included |

**Message queue header examples**:

RabbitMQ:
```python
# Producer
properties = pika.BasicProperties(
    correlation_id=request_id,
    headers={
        "X-Request-ID": request_id,
        "X-Trace-ID": trace_id,
        "X-Source-Service": "order-service",
        "X-Published-At": datetime.utcnow().isoformat() + "Z",
    },
)
channel.basic_publish(exchange="", routing_key="orders", body=body, properties=properties)

# Consumer
def callback(ch, method, properties, body):
    request_id = properties.correlation_id or properties.headers.get("X-Request-ID")
    # Bind to log context...
```

Kafka:
```java
// Producer
ProducerRecord<String, String> record = new ProducerRecord<>("orders", key, value);
record.headers().add("X-Request-ID", requestId.getBytes(StandardCharsets.UTF_8));
record.headers().add("X-Trace-ID", traceId.getBytes(StandardCharsets.UTF_8));
record.headers().add("X-Source-Service", "order-service".getBytes(StandardCharsets.UTF_8));
producer.send(record);

// Consumer
ConsumerRecord<String, String> record = ...;
String requestId = new String(
    record.headers().lastHeader("X-Request-ID").value(),
    StandardCharsets.UTF_8
);
MDC.put("request_id", requestId);
```

AWS SQS:
```python
# Producer
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps(payload),
    MessageAttributes={
        "RequestID": {"DataType": "String", "StringValue": request_id},
        "TraceID": {"DataType": "String", "StringValue": trace_id},
        "SourceService": {"DataType": "String", "StringValue": "order-service"},
    },
)

# Consumer
for message in response["Messages"]:
    request_id = message["MessageAttributes"]["RequestID"]["StringValue"]
    structlog.contextvars.bind_contextvars(request_id=request_id)
```

**Severity guidance for async patterns**:

| Context | Severity |
|---------|----------|
| Messages published without any correlation headers | HIGH |
| Consumer does not extract correlation context from messages | HIGH |
| Correlation breaks at DLQ boundary (DLQ messages lose headers) | MEDIUM |
| Correlation ID format differs between HTTP and message queue headers | LOW |

### Event-Driven Architecture

Event-driven systems (event bus, CQRS/ES, webhooks) require correlation that
spans event chains where a single originating action triggers multiple downstream
events.

**Correlation chain**:
```
User Action -> Event A -> Handler 1 -> Event B -> Handler 2 -> Event C
              [causation_id = null]    [causation_id = A.id]   [causation_id = B.id]
              [correlation_id = X]     [correlation_id = X]    [correlation_id = X]
```

**Requirements**:

| Field | Purpose |
|-------|---------|
| `correlation_id` | Shared across the entire chain; ties all events to the originating action |
| `causation_id` | ID of the immediately preceding event (direct cause); enables event chain reconstruction |
| `event_id` | Unique ID for this specific event; becomes the `causation_id` for events it triggers |
| `source_service` | Which service emitted the event |
| `timestamp` | When the event was produced (ISO 8601 UTC) |

**Event envelope pattern**:
```json
{
  "event_id": "evt_789",
  "event_type": "order.payment_completed",
  "correlation_id": "req_abc123",
  "causation_id": "evt_456",
  "source_service": "payment-service",
  "timestamp": "2025-03-15T14:22:33.456Z",
  "data": {
    "order_id": "ord_123",
    "amount": 99.99,
    "currency": "USD"
  }
}
```

**Severity guidance for event-driven patterns**:

| Context | Severity |
|---------|----------|
| Events carry no correlation_id; chains cannot be reconstructed | HIGH |
| Events have correlation_id but no causation_id (chain ordering lost) | MEDIUM |
| Event envelope does not include source_service or timestamp | MEDIUM |
| Event_id is not globally unique (collision risk) | MEDIUM |
| Full envelope with correlation_id, causation_id, and event_id | INFO (positive) |

### Batch Processing

Batch jobs process collections of items, often triggered by a scheduler or a file
arrival. Correlation must tie the batch invocation, individual item processing,
and any downstream effects together.

**Correlation chain**:
```
Scheduler -> Batch Job -> Item 1 -> Service A
                       -> Item 2 -> Service A
                       -> Item 3 -> Service B
```

**Requirements**:

| Field | Purpose |
|-------|---------|
| `batch_id` | Unique ID for the batch execution; appears in all logs for the batch |
| `batch_item_id` | Identifier for the specific item within the batch |
| `batch_trigger` | How the batch was initiated (scheduled, manual, file_arrival) |
| `correlation_id` | For downstream service calls triggered by item processing |

**Batch logging pattern**:
```python
import uuid
import time

def run_batch(items, batch_trigger="scheduled"):
    batch_id = str(uuid.uuid4())
    structlog.contextvars.bind_contextvars(
        batch_id=batch_id,
        batch_trigger=batch_trigger,
    )

    logger.info("batch_started", item_count=len(items))
    start = time.monotonic()

    results = {"success": 0, "error": 0, "skipped": 0}

    for i, item in enumerate(items):
        item_correlation_id = f"{batch_id}:{item.id}"
        structlog.contextvars.bind_contextvars(
            batch_item_index=i,
            batch_item_id=item.id,
            correlation_id=item_correlation_id,
        )

        try:
            process_item(item)
            results["success"] += 1
        except SkipException:
            results["skipped"] += 1
            logger.info("batch_item_skipped", reason="already_processed")
        except Exception as e:
            results["error"] += 1
            logger.error("batch_item_failed",
                error=str(e),
                error_type=type(e).__name__,
            )

    duration_ms = (time.monotonic() - start) * 1000
    logger.info("batch_completed",
        duration_ms=round(duration_ms, 2),
        **results,
    )
```

**Severity guidance for batch patterns**:

| Context | Severity |
|---------|----------|
| No batch_id; batch failures cannot be isolated to a specific run | HIGH |
| No per-item identification in logs; failed items cannot be identified for retry | MEDIUM |
| Batch downstream calls do not propagate correlation_id | MEDIUM |
| Batch progress not logged (no start/complete/item count) | LOW |
| Full batch correlation with per-item tracking and lifecycle logging | INFO (positive) |

---

## Review Procedure Summary

When evaluating log correlation:

1. **Check request ID generation and extraction**: Verify incoming requests get a correlation ID from headers or a new one is generated. Check for UUID/ULID generation in middleware.
2. **Verify propagation to downstream services**: Confirm outgoing HTTP calls, gRPC calls, and message queue publications include the correlation ID in headers.
3. **Verify inclusion in all logs**: Confirm the correlation ID appears in every log entry during request handling, not just the entry and exit logs.
4. **Check response headers**: Confirm the correlation ID is returned in response headers for client-side debugging.
5. **Audit cross-service propagation**: Verify W3C Trace Context or B3 headers are used consistently across all service boundaries.
6. **Check message queue propagation**: Verify correlation context is included in message headers for async communication.
7. **Evaluate log aggregation**: Confirm centralized log collection, consistent format, timestamp standardization (ISO 8601 UTC), and defined retention.
8. **Assess architecture-specific patterns**: Apply the appropriate checklist from Section 4 based on the system's architecture (synchronous, async, event-driven, batch).
9. **Classify each finding** using the severity tables above, adjusting for system complexity and operational maturity.

---

## Quick Reference: Severity Summary

| Severity | Correlation Findings |
|----------|---------------------|
| CRITICAL | (Correlation issues are not typically CRITICAL alone; they become CRITICAL when combined with security implications such as inability to trace a breach or comply with audit requirements) |
| HIGH | No request correlation in multi-service systems; correlation ID not propagated to downstream services; no centralized log aggregation; messages published without correlation headers; log shipping blocks application on failure; no correlation in event-driven chains |
| MEDIUM | Correlation ID not in log entries; inconsistent header names across services; missing message queue propagation; non-standard timestamp formats; no retention policy; correlation breaks at DLQ boundary; missing causation_id in event chains |
| LOW | Correlation ID not in response headers; custom propagation format instead of W3C; epoch timestamps; missing batch progress logging; aggregation health not monitored |
| INFO | Full correlation chain from ingress through all services; W3C Trace Context consistently used; centralized aggregation with consistent format and retention; complete event envelope with correlation, causation, and event IDs |

---

## References

- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/) -- the standard for distributed context propagation
- [W3C Baggage Specification](https://www.w3.org/TR/baggage/) -- standard for cross-service metadata propagation
- [OpenTelemetry Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/) -- how OTel implements propagation
- [Zipkin B3 Propagation](https://github.com/openzipkin/b3-propagation) -- B3 header specification
- [Google SRE Book, Chapter 12: Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/) -- correlation for incident response
- [Correlation IDs in Microservices](https://www.rapid7.com/blog/post/2016/12/23/the-value-of-correlation-ids/) -- practical guidance
- [12-Factor App: Logs](https://12factor.net/logs) -- logs as event streams
