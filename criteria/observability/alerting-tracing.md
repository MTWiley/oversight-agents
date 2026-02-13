# Alerting and Distributed Tracing Reference Checklist

Canonical reference for evaluating distributed tracing instrumentation, span
quality, trace propagation, sampling strategies, alerting practices, SLO
definitions, health check patterns, and dashboard organization. Complements the
inlined criteria in `review-observability.md`.

Use this checklist during both diff-based and full-repo reviews. Each section maps
to a category in the finding schema. Severity levels follow
`criteria/shared/severity-levels.md`.

---

## 1. Span Creation

### What to Look For

Distributed tracing records the path of a request through a system as a tree of
spans. Every significant operation should produce a span so that operators can
see where time is spent, where errors occur, and how services interact. Missing
spans create blind spots that make latency investigations and root-cause analysis
impossible.

**Operations that require spans**:

| Operation | Why It Needs a Span | Example Span Name |
|-----------|--------------------:|-------------------|
| **Incoming HTTP request** | Entry point into the service; defines the root or server span | `GET /api/orders` |
| **Outgoing HTTP/gRPC call** | Shows inter-service dependencies and downstream latency | `HTTP GET inventory-service` |
| **Database query** | Database calls are the most common source of latency | `SELECT orders` |
| **Message queue publish** | Shows when and where messages are produced | `orders.created publish` |
| **Message queue consume** | Shows processing time and links to the producing span | `orders.created process` |
| **Cache operation** | Cache hits vs. misses affect latency significantly | `redis GET user:123` |
| **External API call** | Third-party latency is outside your control; visibility is critical | `HTTP POST stripe.com/charges` |
| **Significant internal computation** | CPU-bound work that may dominate latency | `render_report` |

**Detection of missing span creation**:
```regex
# HTTP client calls without span context
(?i)(requests\.(get|post|put)|http\.Get|fetch\(|axios\.)(?!.*(?:span|trace|instrument))
# Database queries without tracing
(?i)(execute|query|find|aggregate|cursor)\s*\((?!.*(?:span|trace|instrument))
# Message publishing without span
(?i)(publish|send_message|produce|basic_publish)\s*\((?!.*(?:span|trace|inject))
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No spans created for incoming HTTP requests (no tracing entry point) | HIGH |
| No spans for outgoing HTTP/gRPC calls (downstream latency invisible) | HIGH |
| No spans for database queries in a data-intensive service | HIGH |
| No spans for message queue publish/consume | MEDIUM |
| No spans for cache operations | LOW |
| Missing spans for internal computation that is not a latency concern | LOW |
| Comprehensive span creation covering all significant operations | INFO (positive) |

### Fix Patterns

**Python (OpenTelemetry)**:
```python
from opentelemetry import trace
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor

# Auto-instrumentation covers common libraries
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
Psycopg2Instrumentor().instrument()
RedisInstrumentor().instrument()

# Manual spans for custom operations
tracer = trace.get_tracer(__name__)

def process_order(order_id):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        validate_inventory(order_id)
        charge_payment(order_id)
```

**Go (OpenTelemetry)**:
```go
import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("order-service")

func ProcessOrder(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "process_order",
        trace.WithAttributes(
            attribute.String("order.id", orderID),
        ),
    )
    defer span.End()

    if err := validateInventory(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }
    return chargePayment(ctx, orderID)
}
```

**Node.js (OpenTelemetry)**:
```javascript
const { trace } = require('@opentelemetry/api');

const tracer = trace.getTracer('order-service');

async function processOrder(orderId) {
  return tracer.startActiveSpan('process_order', async (span) => {
    try {
      span.setAttribute('order.id', orderId);
      await validateInventory(orderId);
      await chargePayment(orderId);
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

**Java (OpenTelemetry)**:
```java
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;

@Inject
private Tracer tracer;

public void processOrder(String orderId) {
    Span span = tracer.spanBuilder("process_order")
        .setAttribute("order.id", orderId)
        .startSpan();

    try (Scope scope = span.makeCurrent()) {
        validateInventory(orderId);
        chargePayment(orderId);
    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getMessage());
        throw e;
    } finally {
        span.end();
    }
}
```

---

## 2. Span Attributes and Tags

### What to Look For

Span attributes (also called tags) provide the context that makes traces useful
for debugging. A span with only a name and duration tells you that something
happened and how long it took, but not what happened or why it failed. Attributes
transform spans from timing data into diagnostic data.

**Required attributes by span kind**:

| Span Kind | Required Attributes | Recommended Attributes |
|-----------|--------------------:|----------------------:|
| **HTTP server** | `http.method`, `http.url`, `http.status_code`, `http.route` | `http.request_content_length`, `http.response_content_length`, `user_agent.original`, `client.address` |
| **HTTP client** | `http.method`, `http.url`, `http.status_code` | `http.request_content_length`, `server.address`, `server.port` |
| **Database** | `db.system`, `db.name`, `db.operation` | `db.statement` (sanitized), `db.connection_string` (redacted), `db.sql.table` |
| **gRPC** | `rpc.system`, `rpc.service`, `rpc.method`, `rpc.grpc.status_code` | `rpc.request.metadata.*` |
| **Message queue** | `messaging.system`, `messaging.destination.name`, `messaging.operation` | `messaging.message.id`, `messaging.batch.message_count` |
| **Cache** | `db.system` (e.g., `redis`), `db.operation` | `db.statement`, cache hit/miss indicator |

**OpenTelemetry semantic conventions** (preferred attribute names):

| Convention | Example | Anti-Example |
|-----------|---------|-------------|
| `http.method` | `GET` | `method`, `httpMethod`, `verb` |
| `http.status_code` | `200` | `status`, `statusCode`, `http_code` |
| `db.system` | `postgresql` | `database`, `db_type`, `dbms` |
| `db.statement` | `SELECT * FROM orders WHERE id = ?` | `query`, `sql`, `db_query` |
| `rpc.system` | `grpc` | `protocol`, `rpc_type` |
| `messaging.system` | `rabbitmq` | `broker`, `queue_type`, `mq` |

**Detection of missing or non-standard attributes**:
```regex
# Spans created without attributes
(?i)start_span\s*\(\s*["'][^"']+["']\s*\)(?!\s*\n.*set_attribute)
# Non-standard attribute names
(?i)set_attribute\s*\(\s*["'](?:method|status|database|query|broker)["']
# Database spans without db.system
(?i)start.*span.*(?:query|sql|database|db)(?!.*db\.system)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| HTTP spans missing status code (cannot distinguish success from failure) | HIGH |
| Database spans missing `db.system` and `db.operation` | MEDIUM |
| Spans with no attributes at all (name-only spans) | MEDIUM |
| Non-standard attribute names instead of OTel semantic conventions | LOW |
| Database `db.statement` includes unsanitized user input | HIGH |
| All spans use OTel semantic conventions with comprehensive attributes | INFO (positive) |

### Common Mistakes

**Mistake**: Recording raw SQL with user input in `db.statement`.

Why it is a problem: `db.statement` is sent to the tracing backend and may be
visible to operators, stored in logs, or exposed via API. Unsanitized queries
can leak PII, credentials, or enable information disclosure.

Before (smell):
```python
span.set_attribute("db.statement", f"SELECT * FROM users WHERE email = '{email}'")
```

After (refactored):
```python
span.set_attribute("db.statement", "SELECT * FROM users WHERE email = ?")
span.set_attribute("db.sql.table", "users")
span.set_attribute("db.operation", "SELECT")
```

**Mistake**: Missing error information on failed spans.

Before (smell):
```python
try:
    result = db.execute(query)
except Exception as e:
    span.set_attribute("error", True)  # Insufficient
    raise
```

After (refactored):
```python
try:
    result = db.execute(query)
except Exception as e:
    span.set_status(StatusCode.ERROR, str(e))
    span.record_exception(e)
    span.set_attribute("error.type", type(e).__name__)
    raise
```

---

## 3. Error Recording on Spans

### What to Look For

When an operation fails, the corresponding span must be marked as an error with
sufficient detail to diagnose the problem. Traces where errors are silently
swallowed or marked without detail are nearly useless for debugging.

**Error recording requirements**:

| Requirement | OpenTelemetry API | Purpose |
|-------------|------------------|---------|
| **Set error status** | `span.set_status(StatusCode.ERROR, description)` | Marks the span as failed in the trace UI |
| **Record exception** | `span.record_exception(exception)` | Attaches the exception type, message, and stack trace as a span event |
| **Error attributes** | `span.set_attribute("error.type", ...)` | Enables filtering traces by error type |
| **HTTP status** | `span.set_attribute("http.status_code", 500)` | Standard HTTP error indicator |

**Detection of missing error recording**:
```regex
# Exception handling without span error recording
(?i)except\s+\w+.*:\s*\n(?:(?!set_status|record_exception|RecordError|setStatus).)*raise
# Error returned without span status update (Go)
(?i)if\s+err\s*!=\s*nil\s*\{(?:(?!RecordError|SetStatus).)*return.*err
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Exceptions caught with no error recording on the span (silent failures in traces) | HIGH |
| Error status set but no exception details recorded (no stack trace) | MEDIUM |
| HTTP 5xx responses not reflected as error spans | HIGH |
| HTTP 4xx responses marked as errors (client errors are not server errors) | LOW |
| Comprehensive error recording with status, exception, and error type | INFO (positive) |

### Common Mistakes

**Mistake**: Marking HTTP 4xx responses as span errors.

Why it is a problem: 4xx responses are client errors (bad request, not found,
unauthorized). They indicate the client made a mistake, not that the server
failed. Marking these as errors inflates error rates in trace analysis and
creates noise when searching for actual server failures.

```python
# GOOD - only 5xx are server errors
if response.status_code >= 500:
    span.set_status(StatusCode.ERROR, f"HTTP {response.status_code}")
elif response.status_code >= 400:
    span.set_status(StatusCode.UNSET)  # Not a server error
    span.set_attribute("http.status_code", response.status_code)
```

---

## 4. Trace Propagation

### What to Look For

Trace propagation ensures that spans created in different services are connected
into a single trace. Without propagation, each service generates isolated traces
that cannot be correlated. Propagation works by injecting trace context into
outgoing requests and extracting it from incoming requests.

**Propagation points**:

| Boundary | Injection Point | Extraction Point |
|----------|----------------|-----------------|
| **HTTP** | `traceparent` header on outgoing requests | `traceparent` header on incoming requests |
| **gRPC** | gRPC metadata on outgoing calls | gRPC metadata on incoming calls |
| **Message queue** | Message headers when publishing | Message headers when consuming |
| **Async job** | Job metadata / payload when enqueuing | Job metadata when dequeuing |
| **Thread/goroutine** | Pass `context` object to new thread/goroutine | Receive `context` in new thread/goroutine |

**Trace context standards**:

| Standard | Header(s) | Usage |
|----------|-----------|-------|
| **W3C Trace Context** | `traceparent`, `tracestate` | The standard; use this for new implementations |
| **B3 (Zipkin)** | `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` (multi) or `b3` (single) | Legacy; common in existing Zipkin deployments |
| **Jaeger** | `uber-trace-id` | Legacy Jaeger format; migrate to W3C |
| **AWS X-Ray** | `X-Amzn-Trace-Id` | AWS-specific; OpenTelemetry SDK bridges to this |

**Detection of broken propagation**:
```regex
# HTTP client without context injection
(?i)(requests\.(get|post|put)|http\.NewRequest|fetch\(|axios\.)(?!.*(?:inject|propagat|traceparent|carrier))
# Message publish without context injection
(?i)(basic_publish|send_message|produce)\s*\((?!.*(?:inject|propagat|traceparent|carrier|headers.*trace))
# Async/background work without context propagation
(?i)(go\s+func|Thread\(|threading\.Thread|submit|CompletableFuture)(?!.*(?:ctx|context|span))
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No trace propagation between services (each service has isolated traces) | HIGH |
| HTTP propagation works but message queue propagation is missing | MEDIUM |
| Context lost at async boundaries (goroutines, threads, async jobs) | MEDIUM |
| Using legacy propagation format (B3/Jaeger) with no migration plan | LOW |
| Mixed propagation formats without a composite propagator | MEDIUM |
| W3C Trace Context propagated consistently across all boundaries | INFO (positive) |

### Fix Patterns

**Python -- propagate context via HTTP**:
```python
from opentelemetry import context
from opentelemetry.propagators import inject

def call_downstream(url, payload):
    headers = {}
    inject(headers)  # Injects traceparent and tracestate
    response = requests.post(url, json=payload, headers=headers)
    return response
```

**Python -- propagate context via message queue**:
```python
from opentelemetry import trace, context
from opentelemetry.propagators import inject

def publish_event(channel, exchange, routing_key, body):
    headers = {}
    inject(headers)  # Injects traceparent into headers dict

    properties = pika.BasicProperties(
        headers=headers,
        content_type="application/json",
    )
    channel.basic_publish(
        exchange=exchange,
        routing_key=routing_key,
        body=json.dumps(body),
        properties=properties,
    )

def consume_event(channel, method, properties, body):
    # Extract context from message headers
    ctx = extract(properties.headers)
    token = context.attach(ctx)
    try:
        with tracer.start_as_current_span("process_event", context=ctx):
            handle_event(json.loads(body))
    finally:
        context.detach(token)
```

**Go -- propagate context via HTTP**:
```go
import (
    "net/http"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

func callDownstream(ctx context.Context, url string) (*http.Response, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    // Inject trace context into outgoing headers
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
    return http.DefaultClient.Do(req)
}
```

**Node.js -- propagate context via message queue**:
```javascript
const { context, propagation, trace } = require('@opentelemetry/api');

function publishEvent(channel, queue, message) {
  const headers = {};
  propagation.inject(context.active(), headers);

  channel.sendToQueue(queue, Buffer.from(JSON.stringify(message)), {
    headers,
  });
}

function consumeEvent(msg) {
  const parentCtx = propagation.extract(context.active(), msg.properties.headers);
  const tracer = trace.getTracer('consumer');

  return context.with(parentCtx, () => {
    return tracer.startActiveSpan('process_event', (span) => {
      try {
        handleEvent(JSON.parse(msg.content.toString()));
      } catch (err) {
        span.recordException(err);
        span.setStatus({ code: SpanStatusCode.ERROR });
        throw err;
      } finally {
        span.end();
      }
    });
  });
}
```

---

## 5. Sampling Strategies

### What to Look For

Sampling determines which traces are recorded and exported. Without sampling,
high-traffic services generate enormous volumes of trace data that overwhelm
storage backends and increase costs. With overly aggressive sampling, important
traces (errors, slow requests) are lost.

**Sampling strategies**:

| Strategy | How It Works | Trade-offs |
|----------|-------------|------------|
| **Always-on** | Record every trace | Full visibility; highest cost; suitable for low-traffic or critical services |
| **Head-based (probabilistic)** | Decision made at trace start; downstream services honor the decision | Simple; consistent; may miss rare errors |
| **Tail-based** | Decision made after trace completes based on outcome (error, latency, etc.) | Captures all interesting traces; requires a collector with buffering |
| **Rate-limiting** | Record up to N traces per second | Controls cost; may miss burst patterns |
| **Rule-based** | Different rates for different endpoints or operations | Fine-grained control; more configuration complexity |

**Recommended sampling configuration**:

| Environment | Strategy | Rate |
|-------------|----------|------|
| Development / staging | Always-on | 100% |
| Production (low traffic, < 100 rps) | Always-on or high probability | 100% or 50% |
| Production (medium traffic, 100-1000 rps) | Head-based probabilistic | 1-10% |
| Production (high traffic, > 1000 rps) | Tail-based or rule-based | Tail: keep errors + slow; Head: 0.1-1% |
| Critical paths (payments, auth) | Always-on or tail-based | 100% for errors; 10-100% for success |
| Health checks / readiness probes | Drop or minimal | 0% or 0.01% |

**Detection of sampling problems**:
```regex
# No sampling configuration (defaults to always-on, which may be intentional or an oversight)
(?i)(?:TracerProvider|tracer.*provider|OTEL_TRACES_SAMPLER)
# Health check endpoints not filtered from tracing
(?i)(health|ready|live|ping|heartbeat)(?!.*(?:sampl|filter|exclud|ignore))
# Sampling rate set to zero (no traces at all)
(?i)(sample.*rate|ratio)\s*[:=]\s*0(?:\.0+)?[,\s})]
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No sampling configuration with very high traffic (storage/cost risk) | HIGH |
| Always-on sampling in production at > 1000 rps without tail-based collection | HIGH |
| No traces collected at all (sampling rate = 0 or tracing disabled in production) | HIGH |
| Health check endpoints generating high-volume low-value traces | MEDIUM |
| Head-based sampling that drops error traces (errors not prioritized) | MEDIUM |
| No tail-based sampling for error or high-latency traces | LOW |
| Well-configured sampling with error retention and cost-appropriate rates | INFO (positive) |

### Fix Patterns

**Python -- OpenTelemetry sampler configuration**:
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.sampling import (
    TraceIdRatioBased,
    ParentBasedTraceIdRatio,
)

# Head-based: sample 10% of new traces, honor parent decision for child spans
sampler = ParentBasedTraceIdRatio(0.1)

provider = TracerProvider(sampler=sampler)
trace.set_tracer_provider(provider)
```

**Go -- OpenTelemetry sampler configuration**:
```go
import (
    "go.opentelemetry.io/otel/sdk/trace"
)

// Head-based: sample 10% of new traces
sampler := trace.ParentBased(trace.TraceIDRatioBased(0.1))

tp := trace.NewTracerProvider(
    trace.WithSampler(sampler),
    // ... exporter configuration
)
```

**OpenTelemetry Collector -- tail-based sampling** (recommended for production):
```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      # Always keep error traces
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      # Always keep slow traces (> 2 seconds)
      - name: slow-traces
        type: latency
        latency: {threshold_ms: 2000}
      # Sample 10% of successful traces
      - name: probabilistic-sampling
        type: probabilistic
        probabilistic: {sampling_percentage: 10}
      # Always keep traces for critical paths
      - name: critical-paths
        type: string_attribute
        string_attribute:
          key: http.route
          values: ["/api/payments", "/api/auth/login"]

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [otlp/backend]
```

---

## 6. Span Naming Conventions

### What to Look For

Span names are the primary identifier for operations in trace UIs and analytics.
Good span names are low-cardinality, descriptive, and follow a consistent
pattern. Bad span names produce unreadable trace waterfalls and make grouping
and filtering impossible.

**Naming rules**:

| Rule | Good Example | Bad Example |
|------|-------------|-------------|
| Use the route template, not the resolved URL | `GET /api/users/{id}` | `GET /api/users/12345` |
| Include the HTTP method for HTTP spans | `GET /api/orders` | `/api/orders` |
| Use the operation name for database spans | `SELECT orders` | `db.query` |
| Use `{destination} {operation}` for messaging | `orders.created publish` | `send message` |
| Keep names low cardinality | `process_order` | `process_order_12345` |
| Use descriptive verbs for internal spans | `validate_inventory` | `step_2` |

**Detection of bad span names**:
```regex
# Span names containing entity IDs (high cardinality)
(?i)start_span\s*\(\s*f?["'].*\{.*id.*\}
# Span names that are just HTTP paths with resolved parameters
(?i)start_span\s*\(\s*["'](?:GET|POST|PUT|DELETE)\s+/\w+/\d+
# Generic non-descriptive span names
(?i)start_span\s*\(\s*["'](?:span|operation|task|work|process|handle|do)["']
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Span names contain entity IDs (unbounded cardinality; breaks grouping) | HIGH |
| All spans named generically ("span", "operation", "query") | MEDIUM |
| HTTP span names missing the method verb | LOW |
| Inconsistent naming conventions across services | LOW |
| Descriptive, low-cardinality span names following OTel conventions | INFO (positive) |

---

## 7. OpenTelemetry SDK Configuration

### What to Look For

The OpenTelemetry SDK requires proper initialization to export traces reliably.
Misconfigured SDKs silently drop traces, leak resources, or fail to propagate
context.

**Configuration requirements**:

| Requirement | Description |
|-------------|-------------|
| **TracerProvider** | Initialized at application startup with exporter and sampler |
| **Exporter** | Configured to send traces to a backend (OTLP, Jaeger, Zipkin) |
| **Propagator** | Set to W3C Trace Context (or composite with B3 for interop) |
| **Resource attributes** | `service.name`, `service.version`, `deployment.environment` set |
| **Shutdown hook** | `TracerProvider.shutdown()` called on application exit to flush pending spans |
| **Batch processing** | Spans exported in batches, not individually |

**Detection of SDK misconfiguration**:
```regex
# Missing service.name resource attribute
(?i)TracerProvider(?!.*service\.name)
# Missing shutdown hook
(?i)TracerProvider(?!.*(?:shutdown|atexit|defer.*Shutdown|addShutdownHook))
# No exporter configured
(?i)TracerProvider\(\s*\)(?!.*(?:exporter|Exporter|OTLP|Jaeger|Zipkin))
# No propagator configured
(?i)(?:set_text_map_propagator|SetTextMapPropagator|propagation)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No `service.name` resource attribute (traces unidentifiable in backend) | HIGH |
| No shutdown hook (pending spans lost on graceful shutdown) | MEDIUM |
| No exporter configured (traces generated but never sent) | HIGH |
| No propagator set (defaults may not match the rest of the system) | MEDIUM |
| Missing `deployment.environment` (cannot filter prod vs. staging) | LOW |
| Fully configured SDK with resource attributes, exporter, propagator, and shutdown | INFO (positive) |

### Fix Patterns

**Python -- complete OpenTelemetry setup**:
```python
import atexit
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator

resource = Resource.create({
    "service.name": "order-service",
    "service.version": "1.2.3",
    "deployment.environment": os.environ.get("ENV", "development"),
})

provider = TracerProvider(
    resource=resource,
    sampler=ParentBasedTraceIdRatio(0.1),
)

exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317")
provider.add_span_processor(BatchSpanProcessor(exporter))

trace.set_tracer_provider(provider)
set_global_textmap(CompositePropagator([
    TraceContextTextMapPropagator(),
    W3CBaggagePropagator(),
]))

atexit.register(provider.shutdown)
```

**Go -- complete OpenTelemetry setup**:
```go
package tracing

import (
    "context"
    "os"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func InitTracer(ctx context.Context) (func(), error) {
    res, _ := resource.Merge(
        resource.Default(),
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("order-service"),
            semconv.ServiceVersion("1.2.3"),
            semconv.DeploymentEnvironment(os.Getenv("ENV")),
        ),
    )

    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithResource(res),
        sdktrace.WithBatcher(exporter),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )

    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    shutdown := func() { _ = tp.Shutdown(context.Background()) }
    return shutdown, nil
}
```

**Node.js -- complete OpenTelemetry setup**:
```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION,
        SEMRESATTRS_DEPLOYMENT_ENVIRONMENT } = require('@opentelemetry/semantic-conventions');
const { W3CTraceContextPropagator } = require('@opentelemetry/core');
const { propagation } = require('@opentelemetry/api');

const resource = new Resource({
  [SEMRESATTRS_SERVICE_NAME]: 'order-service',
  [SEMRESATTRS_SERVICE_VERSION]: '1.2.3',
  [SEMRESATTRS_DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
});

const provider = new NodeTracerProvider({ resource });
const exporter = new OTLPTraceExporter({ url: 'http://otel-collector:4317' });
provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

propagation.setGlobalPropagator(new W3CTraceContextPropagator());

process.on('SIGTERM', () => {
  provider.shutdown().then(() => process.exit(0));
});
```

**Java -- complete OpenTelemetry setup (Spring Boot)**:
```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.ResourceAttributes;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.extension.trace.propagation.W3CTraceContextPropagator;

@Configuration
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault().merge(
            Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service",
                ResourceAttributes.SERVICE_VERSION, "1.2.3",
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT,
                    System.getenv().getOrDefault("ENV", "development")
            ))
        );

        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .setSampler(Sampler.parentBased(Sampler.traceIdRatioBased(0.1)))
            .build();

        OpenTelemetrySdk sdk = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()))
            .build();

        Runtime.getRuntime().addShutdownHook(new Thread(tracerProvider::close));
        return sdk;
    }
}
```

---

## 8. Symptom-Based Alerting

### What to Look For

Alerts should fire when users are affected, not when an internal component
misbehaves. Cause-based alerting (e.g., "CPU is high") generates noise because
high CPU does not always affect users. Symptom-based alerting (e.g., "error rate
exceeds SLO") directly correlates with user impact.

**Symptom vs. cause alerting**:

| Symptom (Good) | Cause (Bad) |
|----------------|-------------|
| Error rate > 1% for 5 minutes | Pod restarted |
| p99 latency > 500ms for 5 minutes | CPU utilization > 80% |
| Successful login rate < 99% | Database connection pool at 90% |
| Payment success rate < 99.9% | Disk usage > 75% |
| API availability < 99.95% | Memory usage > 80% |

**When cause-based alerts are acceptable**: Cause-based alerts are appropriate as
secondary signals for capacity planning or as warnings (not pages). For example,
"disk usage > 85%" as a warning ticket is fine. "Disk usage > 85%" as a page at
2 AM is not.

**Detection of cause-based paging alerts**:
```regex
# CPU/memory/disk alerts configured as paging severity
(?i)(cpu|memory|mem|disk|load_average).*(?:critical|page|pager|sev1|P1)
# Pod/container restart alerts as paging
(?i)(restart|oom|OOMKill|CrashLoopBackOff).*(?:critical|page|pager|sev1|P1)
# Infrastructure alerts without user-impact correlation
(?i)alert.*(?:cpu_utilization|memory_usage|disk_usage).*severity.*(?:critical|page)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Only cause-based alerts exist; no symptom-based alerting at all | HIGH |
| Cause-based alerts configured as pages (2 AM wake-ups for CPU spikes) | HIGH |
| Symptom-based alerts exist but thresholds are not tied to SLOs | MEDIUM |
| Cause-based alerts used for capacity planning warnings (non-paging) | INFO (acceptable) |
| Symptom-based alerting tied to SLOs with cause-based as secondary signals | INFO (positive) |

### Prometheus Alert Rule Examples

**Symptom-based: high error rate**:
```yaml
groups:
  - name: slo_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Error rate exceeds 1% SLO"
          description: >
            Error rate is {{ $value | humanizePercentage }} over the last 5 minutes.
            SLO target: 99% success rate.
          runbook: "https://runbooks.internal/high-error-rate"
          dashboard: "https://grafana.internal/d/slo-overview"
```

**Symptom-based: high latency**:
```yaml
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "P99 latency exceeds 500ms SLO"
          description: >
            P99 latency is {{ $value | humanizeDuration }} over the last 5 minutes.
            SLO target: p99 < 500ms.
          runbook: "https://runbooks.internal/high-latency"
```

---

## 9. Alert Quality

### What to Look For

Every alert that fires must be actionable, routed to the right team, and include
enough context for the responder to begin investigation immediately. Low-quality
alerts cause alert fatigue, which causes real alerts to be ignored.

**Alert quality checklist**:

| Quality Criterion | Description |
|-------------------|-------------|
| **Actionable** | Someone can and should do something about this alert right now |
| **Routed** | Alert goes to the team that owns the service, not a shared channel |
| **Deduplicated** | Firing the same alert 100 times does not send 100 pages |
| **Grouped** | Related alerts (same service, same root cause) are grouped together |
| **Runbook linked** | Every paging alert links to a runbook with investigation steps |
| **Dashboard linked** | Alert links to the relevant dashboard with pre-filtered context |
| **Severity appropriate** | Critical = page; Warning = ticket; Info = log |
| **Sustained duration** | Alert requires sustained threshold violation (`for: 5m`), not a single sample |

**Detection of low-quality alerts**:
```regex
# Alert without 'for' duration (fires on single sample)
(?i)alert:.*\n(?:(?!for:).)*expr:
# Alert without runbook annotation
(?i)alert:.*\n(?:(?!runbook).)*annotations:(?:(?!runbook).)*$
# Alert without severity label
(?i)alert:.*\n(?:(?!severity).)*labels:(?:(?!severity).)*$
# Alert without description
(?i)alert:.*\n(?:(?!description).)*annotations:(?:(?!description).)*$
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Paging alerts without runbook links (responder has no guidance) | HIGH |
| Alerts fire on a single sample with no sustained duration requirement | HIGH |
| No alert routing; all alerts go to a single shared channel | MEDIUM |
| No alert deduplication or grouping configured | MEDIUM |
| Alerts without severity classification (all treated equally) | MEDIUM |
| Missing descriptions or context in alert annotations | LOW |
| Well-structured alerts with routing, runbooks, grouping, and appropriate severity | INFO (positive) |

### Alert Fatigue Prevention

**Appropriate thresholds and durations**:

| Alert Type | Bad Threshold | Good Threshold | Reasoning |
|-----------|---------------|----------------|-----------|
| Error rate | > 0% for 1m | > 1% for 5m | Single errors are normal; sustained high rates are not |
| Latency p99 | > 100ms for 1m | > 500ms for 5m | Brief latency spikes are normal; sustained degradation is not |
| CPU usage | > 50% for 1m | > 90% for 15m | CPU spikes are normal; sustained saturation affects users |
| Disk usage | > 50% | > 85% for 30m | Disks are meant to be used; alert only when running out |
| Pod restarts | > 0 for 1m | > 3 in 15m | Single restarts are handled by orchestration; repeated restarts indicate a problem |

**Alertmanager grouping and routing example**:
```yaml
# alertmanager.yml
route:
  receiver: default-slack
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
      group_wait: 10s
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: team-slack
      repeat_interval: 12h

    - match:
        severity: info
      receiver: default-slack
      repeat_interval: 24h

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - routing_key: "<team-routing-key>"
        description: '{{ .CommonAnnotations.summary }}'
        details:
          runbook: '{{ .CommonAnnotations.runbook }}'
          dashboard: '{{ .CommonAnnotations.dashboard }}'

  - name: team-slack
    slack_configs:
      - channel: '#platform-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'

inhibit_rules:
  # If a critical alert is firing, suppress related warnings
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'service']
```

---

## 10. SLIs and SLOs

### What to Look For

Service Level Indicators (SLIs) are quantitative measures of service quality.
Service Level Objectives (SLOs) are targets for those measures. Together, SLIs
and SLOs formalize the contract between a service and its users, and error
budgets derived from SLOs provide a framework for balancing reliability with
feature velocity.

**SLI types**:

| SLI Type | Definition | Example |
|----------|-----------|---------|
| **Request-based (ratio)** | good events / total events | Successful requests / total requests |
| **Window-based (availability)** | good minutes / total minutes in window | Minutes with error rate < 1% / total minutes |

**SLO structure**:

| Component | Example |
|-----------|---------|
| **SLI** | Ratio of successful HTTP responses (status < 500) to total responses |
| **Target** | 99.9% over a 30-day rolling window |
| **Error budget** | 0.1% of requests can fail = 43.2 minutes of downtime per 30 days |
| **Measurement window** | 30-day rolling |
| **Burn rate alerting** | Alert when error budget is being consumed too fast |

**Common SLO targets and their error budgets (30-day window)**:

| SLO Target | Error Budget | Downtime per 30 days | Downtime per year |
|-----------|-------------|---------------------|------------------|
| 99.0% | 1.0% | 7.2 hours | 3.65 days |
| 99.5% | 0.5% | 3.6 hours | 1.83 days |
| 99.9% | 0.1% | 43.2 minutes | 8.77 hours |
| 99.95% | 0.05% | 21.6 minutes | 4.38 hours |
| 99.99% | 0.01% | 4.3 minutes | 52.6 minutes |

**Detection of missing SLO definitions**:
```regex
# Service without SLI/SLO configuration
# (check for absence of SLO-related recording rules or configuration)
(?i)(slo|sli|error.?budget|service.?level)
# Prometheus recording rules for SLIs
(?i)record:.*:sli|record:.*:error_budget|record:.*:burn_rate
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No SLOs defined for a user-facing production service | HIGH |
| SLOs defined but no error budget tracking or alerting | MEDIUM |
| SLI measurement does not match the stated SLO (measuring the wrong thing) | HIGH |
| SLO target set without data (arbitrary number, not based on historical performance) | MEDIUM |
| Error budget burn rate alerts not configured | MEDIUM |
| SLOs defined, measured, and alerted on with error budget tracking | INFO (positive) |

### Error Budget Burn Rate Alerts

Error budget burn rate alerting fires when the error budget is being consumed
faster than expected. This approach catches both sudden outages (fast burn) and
gradual degradation (slow burn).

**Burn rate windows** (from Google SRE Workbook):

| Burn Rate | Window | Budget Consumed | Meaning |
|-----------|--------|----------------|---------|
| 14.4x | 1 hour | 2% | Fast burn; immediate attention needed |
| 6x | 6 hours | 5% | Moderate burn; investigate soon |
| 3x | 1 day | 10% | Slow burn; investigate during business hours |
| 1x | 3 days | 10% | Normal consumption at SLO boundary |

**Prometheus recording rules for SLO tracking**:
```yaml
groups:
  - name: slo_recording_rules
    rules:
      # SLI: ratio of successful requests
      - record: sli:http_requests:success_rate_5m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))

      # Error ratio (1 - SLI)
      - record: sli:http_requests:error_ratio_5m
        expr: |
          1 - sli:http_requests:success_rate_5m

      # Error budget remaining (SLO = 99.9%)
      - record: slo:http_requests:error_budget_remaining
        expr: |
          1 - (
            sli:http_requests:error_ratio_5m / (1 - 0.999)
          )
```

**Burn rate alert rules**:
```yaml
groups:
  - name: slo_burn_rate_alerts
    rules:
      # Fast burn: 2% of 30-day budget consumed in 1 hour
      - alert: SLOFastBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h]))
            /
            sum(rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SLO fast burn: 2% error budget consumed in 1 hour"
          description: >
            Error rate is {{ $value | humanizePercentage }}, which is 14.4x the
            burn rate. At this rate the 30-day error budget will be exhausted
            in {{ printf "%.1f" (divide 100.0 (multiply $value 100.0)) }} hours.
          runbook: "https://runbooks.internal/slo-fast-burn"

      # Slow burn: 5% of 30-day budget consumed in 6 hours
      - alert: SLOSlowBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h]))
            /
            sum(rate(http_requests_total[6h]))
          ) > (6 * 0.001)
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "SLO slow burn: 5% error budget consumed in 6 hours"
          description: >
            Error rate is {{ $value | humanizePercentage }} over the last 6 hours,
            which is 6x the sustainable burn rate.
          runbook: "https://runbooks.internal/slo-slow-burn"
```

---

## 11. Health Check Patterns

### What to Look For

Health checks enable orchestrators (Kubernetes), load balancers, and monitoring
systems to determine whether a service instance can accept traffic. Poor health
checks cause cascading failures (false positives take healthy instances out of
rotation) or invisible outages (false negatives keep broken instances in rotation).

**Health check types**:

| Endpoint | Purpose | When It Should Fail |
|----------|---------|--------------------|
| `/health` or `/healthz` | General health; combines liveness and readiness | When the process is unhealthy |
| `/ready` or `/readyz` | Readiness; can this instance accept traffic? | When dependencies are unavailable or instance is initializing |
| `/live` or `/livez` | Liveness; is the process alive and not deadlocked? | Only when the process is deadlocked or corrupted; NOT for dependency failures |
| `/startup` | Startup probe; has initialization completed? | During initialization only |

**Critical distinction: liveness vs. readiness**:

| Aspect | Liveness (`/live`) | Readiness (`/ready`) |
|--------|-------------------|---------------------|
| **Failure action** | Kill and restart the pod/process | Remove from load balancer; stop sending traffic |
| **Should check dependencies** | NO -- dependency failure should NOT kill the process | YES -- if critical dependencies are down, stop accepting traffic |
| **Should be lightweight** | YES -- must respond quickly, no external calls | May include lightweight dependency checks |
| **Common mistake** | Checking database in liveness (DB outage restarts all pods, causing cascade) | Not checking database in readiness (broken pods receive traffic) |

**Detection of health check problems**:
```regex
# Liveness check that calls external dependencies
(?i)(livez?|liveness|live_check).*(?:db|database|redis|connection|ping|query)
# Health check without structured response
(?i)(healthz?|readyz?|livez?).*(?:return\s+["']ok["']|res\.send\s*\(\s*["']ok["'])
# No health check endpoints defined
(?i)(health|ready|live|healthz|readyz|livez)\s*(?:endpoint|route|path|url)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No health check endpoints in a service deployed on Kubernetes or behind a load balancer | HIGH |
| Liveness check includes dependency checks (database, cache, external API) | HIGH |
| Readiness check does not verify critical dependencies | MEDIUM |
| Health check returns plain text instead of structured JSON | LOW |
| Health check performs heavyweight operations (full database query, large computation) | MEDIUM |
| Health check has no timeout (can hang indefinitely) | MEDIUM |
| Proper liveness/readiness separation with structured responses and dependency isolation | INFO (positive) |

### Fix Patterns

**Structured health check response**:
```json
{
  "status": "healthy",
  "checks": {
    "database": {"status": "healthy", "latency_ms": 2},
    "cache": {"status": "healthy", "latency_ms": 1},
    "message_queue": {"status": "degraded", "message": "high consumer lag"}
  },
  "version": "1.2.3",
  "uptime_seconds": 86400
}
```

**Python (Flask)**:
```python
import time

_start_time = time.monotonic()

@app.route("/live")
def liveness():
    """Liveness: process is alive. No dependency checks."""
    return jsonify({"status": "healthy"}), 200

@app.route("/ready")
def readiness():
    """Readiness: can accept traffic. Checks critical dependencies."""
    checks = {}
    overall_healthy = True

    # Database check with timeout
    try:
        start = time.monotonic()
        db.session.execute(text("SELECT 1"))
        checks["database"] = {
            "status": "healthy",
            "latency_ms": round((time.monotonic() - start) * 1000, 1),
        }
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}
        overall_healthy = False

    # Cache check with timeout
    try:
        start = time.monotonic()
        redis_client.ping()
        checks["cache"] = {
            "status": "healthy",
            "latency_ms": round((time.monotonic() - start) * 1000, 1),
        }
    except Exception as e:
        checks["cache"] = {"status": "unhealthy", "error": str(e)}
        overall_healthy = False

    status_code = 200 if overall_healthy else 503
    return jsonify({
        "status": "healthy" if overall_healthy else "unhealthy",
        "checks": checks,
        "version": os.environ.get("APP_VERSION", "unknown"),
        "uptime_seconds": round(time.monotonic() - _start_time),
    }), status_code

@app.route("/health")
def health():
    """Combined health check."""
    return readiness()
```

**Go**:
```go
func LivenessHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
}

func ReadinessHandler(db *sql.DB, redis *redis.Client) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()

        checks := map[string]any{}
        healthy := true

        // Database
        start := time.Now()
        if err := db.PingContext(ctx); err != nil {
            checks["database"] = map[string]any{"status": "unhealthy", "error": err.Error()}
            healthy = false
        } else {
            checks["database"] = map[string]any{
                "status": "healthy",
                "latency_ms": time.Since(start).Milliseconds(),
            }
        }

        // Redis
        start = time.Now()
        if err := redis.Ping(ctx).Err(); err != nil {
            checks["cache"] = map[string]any{"status": "unhealthy", "error": err.Error()}
            healthy = false
        } else {
            checks["cache"] = map[string]any{
                "status": "healthy",
                "latency_ms": time.Since(start).Milliseconds(),
            }
        }

        status := http.StatusOK
        statusStr := "healthy"
        if !healthy {
            status = http.StatusServiceUnavailable
            statusStr = "unhealthy"
        }

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(status)
        json.NewEncoder(w).Encode(map[string]any{
            "status": statusStr,
            "checks": checks,
        })
    }
}
```

**Kubernetes probe configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: order-service
          livenessProbe:
            httpGet:
              path: /live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3    # Restart after 3 consecutive failures
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2    # Remove from LB after 2 consecutive failures
          startupProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 30   # Allow up to 150s for startup
```

---

## 12. Dashboard Organization

### What to Look For

Dashboards are the primary tool operators use to understand system state during
incidents and for routine monitoring. Poorly organized dashboards slow down
incident response and lead to incorrect conclusions.

**Dashboard hierarchy**:

| Level | Purpose | Content |
|-------|---------|---------|
| **L0: Business overview** | "Is the business working?" | Key business metrics (orders/min, revenue, active users), overall SLO status |
| **L1: Service overview** | "Which service has a problem?" | RED metrics per service, SLO burn rates, inter-service dependencies |
| **L2: Service detail** | "What is wrong with this service?" | Per-endpoint RED, USE metrics (pools, queues), error breakdown, recent deployments |
| **L3: Infrastructure** | "What resource is constrained?" | CPU, memory, disk, network per host/pod, container metrics |

**Dashboard quality requirements**:

| Requirement | Description |
|-------------|-------------|
| **Variables/filters** | Dashboards use template variables for environment, service, instance (not hardcoded values) |
| **Thresholds** | Panels show threshold lines for SLO targets (e.g., red line at 500ms latency target) |
| **Units** | Every panel has correct units configured (seconds not "value", bytes not "count") |
| **Time range** | Default time range appropriate for the dashboard level (L0: 24h, L2: 1h) |
| **Annotations** | Deployment markers and incident markers shown on time-series graphs |
| **Links** | Drill-down links from L0 to L1 to L2 dashboards |
| **No vanity metrics** | Every panel answers a question an operator would ask during an incident |

**Detection of dashboard problems**:
```regex
# Hardcoded service names instead of variables
(?i)(datasource|target|expr).*["'](?:payment-service|order-service|user-service)["'](?!.*\$)
# Missing units on panels
(?i)"unit"\s*:\s*(?:"none"|"short"|"")
# Dashboard without variables
(?i)"templating"\s*:\s*\{\s*"list"\s*:\s*\[\s*\]
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No dashboards for a production service | HIGH |
| Dashboards exist but have no SLO threshold lines (operators cannot see target violations) | MEDIUM |
| All panels hardcoded to a single environment or service (no variables) | MEDIUM |
| Missing units on panels (ambiguous data) | LOW |
| No deployment annotations on time-series graphs | LOW |
| No drill-down links between dashboard levels | LOW |
| Dashboard without any RED or USE metrics (only vanity metrics) | MEDIUM |
| Well-organized hierarchy with variables, thresholds, units, and drill-down links | INFO (positive) |

### Dashboard Panel Recommendations

**L1 Service Overview -- essential panels**:

| Panel | Query Pattern (Prometheus) | Purpose |
|-------|---------------------------|---------|
| Request rate | `sum(rate(http_requests_total{service="$service"}[5m]))` | Current traffic |
| Error rate | `sum(rate(http_requests_total{service="$service",status=~"5.."}[5m])) / sum(rate(http_requests_total{service="$service"}[5m]))` | Current reliability |
| p50/p95/p99 latency | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))` | Latency distribution |
| Error budget remaining | `1 - (error_ratio / (1 - 0.999))` | SLO status |
| Active connections | `db_connection_pool_active{service="$service"}` | Resource utilization |
| Recent deployments | Annotation query on deployment events | Change correlation |

---

## Review Procedure Summary

When evaluating alerting and distributed tracing:

1. **Check span creation**: Verify spans exist for incoming requests, outgoing calls, database queries, message queue operations, and cache operations. Flag significant operations without spans.
2. **Audit span attributes**: Confirm spans include semantic convention attributes (`http.method`, `http.status_code`, `db.system`, etc.). Flag name-only spans and non-standard attribute names.
3. **Verify error recording**: Confirm exceptions set span status to ERROR, record the exception, and include error type attributes. Flag silent failures in traces.
4. **Check trace propagation**: Verify trace context is propagated across all service boundaries (HTTP, gRPC, message queues, async jobs). Flag broken propagation chains.
5. **Evaluate sampling strategy**: Confirm sampling is configured appropriately for traffic volume. Flag always-on sampling at high traffic and missing error retention.
6. **Audit span naming**: Check for low-cardinality, descriptive span names. Flag names containing entity IDs or generic names.
7. **Verify SDK configuration**: Confirm `service.name` resource attribute, exporter, propagator, and shutdown hook are configured. Flag missing elements.
8. **Evaluate alerting philosophy**: Confirm alerts are symptom-based and tied to user impact. Flag cause-based paging alerts.
9. **Audit alert quality**: Confirm alerts have sustained duration requirements, runbook links, severity labels, routing, and deduplication. Flag alerts that fire on single samples without runbooks.
10. **Check SLO definitions**: Confirm SLIs are defined, SLO targets are set, error budgets are tracked, and burn rate alerts are configured. Flag user-facing services without SLOs.
11. **Evaluate health checks**: Confirm liveness and readiness probes are separate, liveness does not check dependencies, and responses are structured JSON. Flag dependency checks in liveness probes.
12. **Assess dashboards**: Confirm dashboard hierarchy exists with variables, thresholds, units, and drill-down links. Flag hardcoded values and missing SLO threshold lines.
13. **Classify each finding** using the severity tables above, adjusting for the service's criticality and architectural complexity.

---

## Quick Reference: Severity Summary

| Severity | Alerting and Tracing Findings |
|----------|------------------------------|
| CRITICAL | (Tracing/alerting issues are not typically CRITICAL alone; they become CRITICAL when combined with an active incident where lack of tracing or alerting directly extends the outage or prevents root-cause identification) |
| HIGH | No spans for incoming requests or outgoing calls; HTTP spans missing status code; no trace propagation between services; always-on sampling at very high traffic with no collection; no tracing at all in production; only cause-based paging alerts; paging alerts without runbooks; alerts firing on single samples; no SLOs for user-facing services; SLI measures the wrong thing; no health checks on orchestrated services; liveness probe checks external dependencies; no dashboards for production services; `db.statement` with unsanitized user input; exceptions not recorded on spans; HTTP 5xx not reflected as error spans; span names containing entity IDs; no `service.name` resource attribute; no exporter configured |
| MEDIUM | No spans for message queue operations; name-only spans without attributes; non-standard attribute naming; context lost at async boundaries; mixed propagation formats; head-based sampling dropping errors; health check with no timeout; heavyweight health checks; readiness not checking dependencies; symptom alerts not tied to SLOs; no error budget tracking; no alert routing; no deduplication or grouping; alerts without severity; hardcoded dashboard values; no SLO thresholds on dashboards; dashboards with only vanity metrics; missing propagator configuration; no shutdown hook; error status set without exception details |
| LOW | No spans for cache operations; HTTP span names missing method verb; inconsistent naming across services; legacy propagation format with no migration plan; no tail-based sampling for errors; health check returning plain text; missing dashboard units; no deployment annotations; no drill-down links; missing `deployment.environment`; 4xx marked as span errors |
| INFO | Comprehensive span creation; OTel semantic conventions used; full error recording with status, exception, and type; W3C Trace Context across all boundaries; well-configured sampling with error retention; descriptive low-cardinality span names; fully configured SDK; symptom-based alerting tied to SLOs; alerts with runbooks, routing, grouping, and sustained duration; SLOs with error budget tracking and burn rate alerts; proper liveness/readiness separation; structured health responses; well-organized dashboard hierarchy with variables, thresholds, and drill-down |

---

## References

- [OpenTelemetry Tracing Specification](https://opentelemetry.io/docs/specs/otel/trace/) -- authoritative reference for span creation, attributes, and context propagation
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/) -- standard attribute names for HTTP, database, messaging, and RPC spans
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/) -- the standard for distributed trace context propagation
- [Zipkin B3 Propagation](https://github.com/openzipkin/b3-propagation) -- B3 header specification for Zipkin ecosystem interop
- [Google SRE Book, Chapter 4: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) -- SLI, SLO, and error budget fundamentals
- [Google SRE Workbook, Chapter 5: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) -- burn rate alerting methodology
- [Google SRE Book, Chapter 6: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) -- the four golden signals and symptom-based alerting
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) -- alert rule syntax and configuration
- [Prometheus Recording Rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/) -- pre-computed query results for SLO tracking
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/) -- routing, grouping, and inhibition
- [OpenTelemetry Collector Tail Sampling Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor) -- tail-based sampling configuration
- [Kubernetes Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) -- probe configuration reference
