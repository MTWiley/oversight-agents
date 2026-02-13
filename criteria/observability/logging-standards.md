# Logging Standards Reference Checklist

Canonical reference for evaluating logging quality, structured logging adoption,
log level usage, sensitive data exposure, and language-specific patterns. Complements
the inlined criteria in `review-observability.md`.

Use this checklist during both diff-based and full-repo reviews. Each section maps
to a category in the finding schema. Severity levels follow
`criteria/shared/severity-levels.md`.

---

## 1. Structured Logging

### What to Look For

Structured logging produces machine-parseable output (JSON, logfmt, or key-value
pairs) that log aggregation systems can index, query, and alert on. Unstructured
free-text logs are the single biggest obstacle to effective observability.

**Signs of unstructured logging**:
```regex
# Python print statements used for logging
(?i)print\s*\(\s*f?["'].*(?:error|warn|debug|info|fail|exception)
# Console.log with string concatenation instead of structured fields
console\.log\s*\(\s*["'`].*\+
# Go fmt.Println / fmt.Printf used instead of structured logger
fmt\.(?:Print|Printf|Println)\s*\(
# Java System.out / System.err
System\.(out|err)\.print
```

**Detection: JSON structured logging present**:
```regex
# Python structlog or JSON formatter
structlog\.get_logger|structlog\.configure|JSONRenderer|json_format
# Go slog / zerolog / zap
slog\.New|slog\.With|zerolog\.New|zap\.NewProduction|zap\.NewDevelopment
# Node.js pino / winston JSON
pino\(|createLogger.*format.*json|winston\.createLogger
# Java logback JSON encoder / Log4j2 JSON layout
JsonLayout|JsonEncoder|JSONLayout|StructuredArguments
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No structured logging in a service that processes requests (all free-text) | HIGH |
| Mixed structured and unstructured logging within the same service | MEDIUM |
| `print()` or `console.log()` used for operational logging in production code | MEDIUM |
| Structured logging present but inconsistent field names across modules | MEDIUM |
| `fmt.Println` for debug output left in committed code | LOW |
| Well-configured structured logging with consistent field conventions | INFO (positive) |

### Common Mistakes

**Mistake**: Logging with string interpolation instead of structured fields.

Why it is a problem: String interpolation bakes variable values into the message
string, making it impossible to filter or aggregate by field values. Every unique
message becomes a separate log pattern.

Before (smell):
```python
import logging
logger = logging.getLogger(__name__)

# BAD - values embedded in message string
logger.info(f"User {user_id} placed order {order_id} for ${amount}")
logger.error(f"Failed to process payment for order {order_id}: {str(e)}")
```

After (refactored):
```python
import structlog
logger = structlog.get_logger()

# GOOD - values as structured fields
logger.info("order_placed", user_id=user_id, order_id=order_id, amount=amount)
logger.error("payment_failed", order_id=order_id, error=str(e), error_type=type(e).__name__)
```

**Mistake**: Using Go `fmt.Printf` or `log.Printf` instead of a structured logger.

Before (smell):
```go
// BAD - unstructured, unparseable
log.Printf("Processing request %s from user %s", requestID, userID)
log.Printf("ERROR: database query failed: %v", err)
```

After (refactored):
```go
// GOOD - structured with slog
logger := slog.Default()
logger.Info("processing_request",
    slog.String("request_id", requestID),
    slog.String("user_id", userID),
)
logger.Error("database_query_failed",
    slog.String("error", err.Error()),
    slog.String("request_id", requestID),
)
```

**Mistake**: Node.js `console.log` with string concatenation in production code.

Before (smell):
```javascript
// BAD - unparseable, no level, no fields
console.log("Processing order " + orderId + " for user " + userId);
console.log("ERROR: " + err.message);
```

After (refactored):
```javascript
// GOOD - structured with pino
const logger = require('pino')();

logger.info({ orderId, userId }, 'processing_order');
logger.error({ orderId, err: err.message, stack: err.stack }, 'order_processing_failed');
```

---

## 2. Standard Field Set

### What to Look For

Every log entry should include a baseline set of fields that enable filtering,
correlation, and triage. Missing standard fields make logs significantly harder
to use during incident response.

**Required fields** (severity escalates with omission):

| Field | Purpose | Example Value |
|-------|---------|---------------|
| `timestamp` | When the event occurred (ISO 8601 UTC) | `2025-03-15T14:22:33.456Z` |
| `level` | Severity of the event | `info`, `warn`, `error`, `debug` |
| `message` | Human-readable event description | `order_placed` |
| `service` | Name of the emitting service | `payment-service` |
| `request_id` | Unique identifier for the request | `req_a1b2c3d4` |
| `trace_id` | Distributed trace identifier | `4bf92f3577b34da6a3ce929d0e0e4736` |

**Recommended fields** (contextual, not always required):

| Field | Purpose | Example Value |
|-------|---------|---------------|
| `span_id` | Current span within a trace | `00f067aa0ba902b7` |
| `user_id` | Authenticated user performing the action | `usr_12345` |
| `environment` | Deployment environment | `production`, `staging` |
| `version` | Service version or commit SHA | `v2.3.1`, `abc123f` |
| `duration_ms` | Operation duration in milliseconds | `142` |
| `error_type` | Exception or error class name | `ConnectionTimeout` |
| `component` | Subsystem within the service | `database`, `cache`, `auth` |

**Detection of missing standard fields**:
```regex
# Logger initialization without service name configuration
(?i)(logging\.basicConfig|getLogger)\s*\(\s*\)
# Structured log call without timestamp field (timestamp should come from formatter)
# Logger missing request_id in web request handlers
(?i)def\s+(get|post|put|delete|patch|handle)\w*\(.*\):[^}]*logger\.(info|warn|error)(?!.*request_id)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No timestamp in log output | HIGH |
| No log level in output (all messages at same level or level absent) | HIGH |
| No service name in multi-service architecture | HIGH |
| No request_id in HTTP request handlers | MEDIUM |
| No trace_id in distributed system | MEDIUM |
| Missing environment or version fields | LOW |
| All standard fields present and consistent | INFO (positive) |

### Fix Patterns

**Python structlog with standard fields**:
```python
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

# Bind service-level context once at startup
logger = structlog.get_logger().bind(
    service="payment-service",
    environment=os.environ.get("ENV", "development"),
    version=os.environ.get("APP_VERSION", "unknown"),
)
```

**Go slog with standard fields**:
```go
import (
    "log/slog"
    "os"
)

func NewLogger(serviceName, version string) *slog.Logger {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    return slog.New(handler).With(
        slog.String("service", serviceName),
        slog.String("version", version),
        slog.String("environment", os.Getenv("ENV")),
    )
}
```

**Node.js pino with standard fields**:
```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: 'payment-service',
    version: process.env.APP_VERSION || 'unknown',
    environment: process.env.NODE_ENV || 'development',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});
```

**Java slf4j with MDC for standard fields**:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

public class RequestFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        try {
            MDC.put("service", "payment-service");
            MDC.put("request_id", generateRequestId());
            MDC.put("trace_id", extractTraceId(req));
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 3. Log Level Usage

### What to Look For

Log levels exist to control signal-to-noise ratio. Incorrect log levels cause
operators to either miss critical information or drown in noise. Each level has
a specific contract.

**Level definitions and usage**:

| Level | Contract | Examples |
|-------|----------|----------|
| **ERROR** | Something failed that requires attention. An operation could not complete. | Database connection lost, payment processing failed, unhandled exception, external API returned 5xx after retries exhausted |
| **WARN** | Something unexpected happened but the operation continued. May indicate a future problem. | Retry attempt succeeded, fallback path taken, deprecated API called, rate limit approaching threshold, certificate expiring in 7 days |
| **INFO** | Normal operational events. The "what happened" log for business operations. | Request received, order placed, user authenticated, deployment started, configuration loaded, scheduled job completed |
| **DEBUG** | Detailed diagnostic information for development and troubleshooting. Not enabled in production by default. | SQL query text, HTTP request/response bodies, cache hit/miss, algorithm step details, variable state during processing |

**Detection of incorrect log levels**:
```regex
# ERROR for non-errors (expected conditions logged as ERROR)
(?i)logger\.error.*(?:not found|no results|empty|already exists|duplicate|cache miss)
# INFO for debugging detail (too verbose for INFO)
(?i)logger\.info.*(?:query|sql|request body|response body|payload|entering|exiting|step \d)
# DEBUG for operational events (too important for DEBUG)
(?i)logger\.debug.*(?:started|completed|deployed|authenticated|order|payment|created|deleted)
# WARN for normal operations (crying wolf)
(?i)logger\.warn.*(?:successfully|completed|started|loaded|ready)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Errors logged at INFO or DEBUG level (hiding failures) | HIGH |
| Normal operations logged at ERROR level (alert fatigue) | MEDIUM |
| Excessive INFO logging that should be DEBUG (log volume) | MEDIUM |
| Debug-level detail at INFO in production | MEDIUM |
| Missing ERROR log for exception handling (silent failures) | HIGH |
| Inconsistent level usage across modules for similar events | LOW |
| Well-documented level usage following a consistent contract | INFO (positive) |

### Common Mistakes

**Mistake**: Logging expected business conditions at ERROR level.

Why it is a problem: When "user not found" generates an ERROR log, and this happens
hundreds of times per hour for legitimate 404 responses, the ERROR log becomes
meaningless noise. Real errors (database down, payment provider unreachable) are
buried in false alarms. Alert fatigue follows.

Before (smell):
```python
def get_user(user_id):
    user = db.users.find_one({"id": user_id})
    if not user:
        logger.error("User not found", user_id=user_id)  # BAD - expected condition
        raise NotFoundError("User not found")
    return user
```

After (refactored):
```python
def get_user(user_id):
    user = db.users.find_one({"id": user_id})
    if not user:
        logger.info("user_not_found", user_id=user_id)  # GOOD - expected condition
        raise NotFoundError("User not found")
    return user
```

**Mistake**: Swallowing exceptions without logging them.

Before (smell):
```python
try:
    result = external_api.call(payload)
except Exception:
    pass  # BAD - silent failure
```

After (refactored):
```python
try:
    result = external_api.call(payload)
except Exception as e:
    logger.error("external_api_call_failed",
        error=str(e),
        error_type=type(e).__name__,
        payload_id=payload.get("id"),
    )
    raise
```

**Mistake**: Logging sensitive request/response bodies at INFO level.

Before (smell):
```python
logger.info("API request", body=json.dumps(request.json))  # BAD - may contain PII
logger.info("API response", body=json.dumps(response.json()))  # BAD - verbose at INFO
```

After (refactored):
```python
logger.debug("api_request_body", body=redact_sensitive_fields(request.json))
logger.info("api_request_completed",
    method=request.method,
    path=request.path,
    status=response.status_code,
    duration_ms=elapsed_ms,
)
```

---

## 4. Log Content Quality

### What to Look For

A log message should provide enough context for an operator to understand what
happened, why it matters, and what to investigate next -- without requiring access
to the source code.

**Quality criteria**:

| Criterion | Good Example | Bad Example |
|-----------|-------------|-------------|
| **Context**: What was happening when the event occurred | `"payment_processing_failed", order_id="ord_123", provider="stripe"` | `"Error occurred"` |
| **Error details**: Full error type, message, and relevant state | `"db_connection_failed", error="connection refused", host="db-primary.internal", retry_count=3` | `"Database error"` |
| **Operation lifecycle**: Start/end with duration for important operations | `"order_processing_completed", order_id="ord_123", duration_ms=342, item_count=5` | `"Done"` |
| **Identifiers**: All relevant entity IDs for cross-referencing | `"invoice_generated", order_id="ord_123", invoice_id="inv_456", user_id="usr_789"` | `"Invoice generated"` |
| **Action taken**: What the system did in response | `"circuit_breaker_opened", service="payment-api", failure_count=10, cooldown_seconds=30` | `"Too many failures"` |

**Detection of low-quality log messages**:
```regex
# Single-word or very short messages with no context
logger\.(info|warn|error|debug)\s*\(\s*["'][\w\s]{1,15}["']\s*\)
# Generic error messages
logger\.error\s*\(\s*["'](?:error|failed|exception|something went wrong|error occurred)["']
# Missing error details in exception handler
except\s+\w+(?:\s+as\s+\w+)?:\s*\n\s*logger\.\w+\s*\(\s*["'][^"']*["']\s*\)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Exception caught and logged with no error details (type, message, context) | HIGH |
| Log messages with no identifiers in request handling code | MEDIUM |
| Generic messages like "error occurred" with no context | MEDIUM |
| Operations with no lifecycle logging (no start/complete or duration) for critical paths | MEDIUM |
| Log messages that duplicate information already in the structured fields | LOW |
| Rich contextual logging with identifiers, durations, and error details | INFO (positive) |

### Fix Patterns

**Error logging with full context**:

Before (smell):
```go
if err != nil {
    log.Println("error processing request")  // useless
    return err
}
```

After (refactored):
```go
if err != nil {
    logger.Error("request_processing_failed",
        slog.String("request_id", requestID),
        slog.String("user_id", userID),
        slog.String("error", err.Error()),
        slog.String("operation", "create_order"),
        slog.Int("retry_count", retryCount),
    )
    return fmt.Errorf("processing request %s: %w", requestID, err)
}
```

**Operation lifecycle logging**:

```python
import time

def process_batch(batch_id, items):
    logger.info("batch_processing_started",
        batch_id=batch_id,
        item_count=len(items),
    )
    start = time.monotonic()

    success_count = 0
    error_count = 0
    for item in items:
        try:
            process_item(item)
            success_count += 1
        except Exception as e:
            error_count += 1
            logger.error("batch_item_failed",
                batch_id=batch_id,
                item_id=item.id,
                error=str(e),
                error_type=type(e).__name__,
            )

    duration_ms = (time.monotonic() - start) * 1000
    logger.info("batch_processing_completed",
        batch_id=batch_id,
        item_count=len(items),
        success_count=success_count,
        error_count=error_count,
        duration_ms=round(duration_ms, 2),
    )
```

---

## 5. Sensitive Data in Logs

### What to Look For

Logs frequently end up in centralized systems with broad access. Sensitive data
in logs violates privacy regulations (GDPR, HIPAA, PCI-DSS), creates security
risks if log storage is compromised, and may expose credentials that enable
further attacks.

**Sensitive data categories and detection patterns**:

| Category | Detection Regex | Examples |
|----------|----------------|----------|
| **Passwords** | `(?i)(password\|passwd\|pwd)\s*[:=]\s*["']?[^\s"',;]{4,}` | `password=secret123` |
| **API keys/tokens** | `(?i)(api[_-]?key\|token\|bearer\|authorization)\s*[:=]\s*["']?[A-Za-z0-9\-._]{20,}` | `api_key=sk_live_abc123...` |
| **Credit card numbers** | `\b[3-6]\d{3}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b` | `4111-1111-1111-1111` |
| **Social Security Numbers** | `\b\d{3}[-]?\d{2}[-]?\d{4}\b` | `123-45-6789` |
| **Email addresses** (PII context) | `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z\|a-z]{2,}\b` | `user@example.com` |
| **Phone numbers** | `\b\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b` | `+1-555-123-4567` |
| **IP addresses** (internal) | `\b(?:10\|172\.(?:1[6-9]\|2\d\|3[01])\|192\.168)\.\d{1,3}\.\d{1,3}\b` | `10.0.1.42` |
| **JWT tokens** | `eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}` | `eyJhbG...` |
| **Connection strings** | `(?i)(postgres\|mysql\|mongodb\|redis)://[^:]+:[^@]+@` | `postgres://user:pass@host/db` |
| **AWS keys** | `AKIA[0-9A-Z]{16}` | `AKIAIOSFODNN7EXAMPLE` |

**Code patterns that commonly leak sensitive data**:
```regex
# Logging entire request objects (may contain auth headers, body with PII)
logger\.\w+\(.*(?:request|req)\s*\)
logger\.\w+\(.*(?:headers|cookies)\s*[,)]
# Logging entire user objects (may contain password hashes, PII)
logger\.\w+\(.*(?:user|customer|patient|account)\s*[,)]
# Logging entire exception with args (may contain sensitive function arguments)
logger\.\w+\(.*(?:repr|str)\s*\(\s*(?:exc|exception|error|e)\s*\)
# Java toString logging of objects
logger\.\w+\(.*\.toString\(\)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Passwords, tokens, or API keys logged in plaintext | CRITICAL |
| Credit card numbers or SSN appearing in log output | CRITICAL |
| Full connection strings with credentials in logs | CRITICAL |
| PII (email, phone, name) logged without redaction in a regulated system | HIGH |
| Internal IP addresses or infrastructure details in externally accessible logs | MEDIUM |
| Entire request/response objects logged without field filtering | MEDIUM |
| Logging full user objects that may contain sensitive fields | MEDIUM |
| Sensitive data redaction implemented consistently | INFO (positive) |

### Common Mistakes

**Mistake**: Logging entire request objects including authorization headers.

Before (smell):
```python
logger.info("request_received", headers=dict(request.headers))
# Output includes: Authorization: Bearer eyJ...
```

After (refactored):
```python
REDACTED_HEADERS = {"authorization", "cookie", "x-api-key", "proxy-authorization"}

def safe_headers(headers):
    return {
        k: "[REDACTED]" if k.lower() in REDACTED_HEADERS else v
        for k, v in headers.items()
    }

logger.info("request_received", headers=safe_headers(request.headers))
```

**Mistake**: Logging user objects that contain password hashes or PII.

Before (smell):
```python
logger.info("user_created", user=user.__dict__)
# Output includes: password_hash, email, phone, ssn
```

After (refactored):
```python
logger.info("user_created",
    user_id=user.id,
    role=user.role,
    created_at=user.created_at.isoformat(),
)
```

**Mistake**: Error messages that include function arguments with sensitive data.

Before (smell):
```python
def authenticate(username, password):
    try:
        result = auth_service.login(username, password)
    except AuthError as e:
        logger.error(f"Auth failed: {e}")  # e.args may contain password
        raise
```

After (refactored):
```python
def authenticate(username, password):
    try:
        result = auth_service.login(username, password)
    except AuthError as e:
        logger.error("authentication_failed",
            username=username,
            error_type=type(e).__name__,
            error_code=getattr(e, 'code', None),
        )
        raise
```

### Redaction Patterns

**Python structlog processor for sensitive field redaction**:
```python
import re

SENSITIVE_PATTERNS = {
    "password": re.compile(r".*"),
    "token": re.compile(r".*"),
    "api_key": re.compile(r".*"),
    "authorization": re.compile(r".*"),
    "credit_card": re.compile(r"\d{4}"),
    "ssn": re.compile(r"\d{3}"),
}

SENSITIVE_VALUE_PATTERNS = [
    re.compile(r"(?i)(password|secret|token|key)\s*[:=]\s*\S+"),
    re.compile(r"\b[3-6]\d{3}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b"),
    re.compile(r"eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}"),
]

def redact_sensitive_fields(logger, method_name, event_dict):
    for key in list(event_dict.keys()):
        key_lower = key.lower()
        for sensitive_key in SENSITIVE_PATTERNS:
            if sensitive_key in key_lower:
                event_dict[key] = "[REDACTED]"
                break
        else:
            if isinstance(event_dict[key], str):
                for pattern in SENSITIVE_VALUE_PATTERNS:
                    if pattern.search(event_dict[key]):
                        event_dict[key] = pattern.sub("[REDACTED]", event_dict[key])
    return event_dict
```

**Go middleware for sensitive field redaction**:
```go
// RedactingHandler wraps an slog.Handler to redact sensitive fields.
type RedactingHandler struct {
    inner          slog.Handler
    sensitiveKeys  map[string]bool
}

var defaultSensitiveKeys = map[string]bool{
    "password": true, "token": true, "api_key": true,
    "secret": true, "authorization": true, "credit_card": true,
    "ssn": true, "cookie": true,
}

func (h *RedactingHandler) Handle(ctx context.Context, r slog.Record) error {
    redacted := slog.NewRecord(r.Time, r.Level, r.Message, r.PC)
    r.Attrs(func(a slog.Attr) bool {
        if h.sensitiveKeys[strings.ToLower(a.Key)] {
            redacted.AddAttrs(slog.String(a.Key, "[REDACTED]"))
        } else {
            redacted.AddAttrs(a)
        }
        return true
    })
    return h.inner.Handle(ctx, redacted)
}
```

---

## 6. Log Volume Management

### What to Look For

Uncontrolled log volume increases costs (storage, ingestion, egress), degrades
query performance, and can overwhelm log aggregation pipelines. Effective log
volume management balances observability needs with operational costs.

**Volume control mechanisms**:

| Mechanism | Purpose | Implementation |
|-----------|---------|----------------|
| **Level configuration** | Adjust verbosity per environment | DEBUG in dev, INFO in production |
| **Per-module levels** | Noisy modules at WARN, critical paths at DEBUG | Logger hierarchy configuration |
| **Sampling** | Reduce volume for high-frequency events | Log 1 in N for health checks, high-traffic endpoints |
| **Rate limiting** | Prevent log storms from filling disks | Max N messages per second per category |
| **Size limits** | Prevent individual log entries from being oversized | Truncate fields over N bytes |
| **Conditional debug** | Enable DEBUG for specific requests/users | Feature flag or header-based debug activation |

**Detection of volume problems**:
```regex
# DEBUG level enabled in production configuration
(?i)(level|log_level|LOG_LEVEL)\s*[:=]\s*["']?debug
# Logging inside tight loops without sampling
for\s.*:\s*\n(\s+.*\n)*\s+logger\.(info|warn|debug)
# Logging entire payloads at INFO (verbose)
logger\.info\(.*(?:payload|body|data|content|response|request)\s*=
# No log level configuration (hardcoded)
logging\.basicConfig\(\s*\)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| DEBUG level enabled in production with no mechanism to change it | HIGH |
| Logging inside tight loops with no sampling (potential log storm) | HIGH |
| No externalized log level configuration (requires code change to adjust) | MEDIUM |
| Large payloads logged at INFO level in high-traffic paths | MEDIUM |
| No log rotation or retention policy for file-based logging | MEDIUM |
| Missing sampling for high-frequency events (health checks, heartbeats) | LOW |
| Proper level configuration with environment-based defaults | INFO (positive) |

### Fix Patterns

**Environment-based log level configuration**:

```python
# Python - externalized log level
import os
import logging

LOG_LEVEL = os.environ.get("LOG_LEVEL", "INFO").upper()
logging.basicConfig(level=getattr(logging, LOG_LEVEL))

# Per-module override
logging.getLogger("urllib3").setLevel(logging.WARNING)
logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)
```

```go
// Go - externalized log level with slog
func parseLogLevel(level string) slog.Level {
    switch strings.ToUpper(level) {
    case "DEBUG":
        return slog.LevelDebug
    case "WARN", "WARNING":
        return slog.LevelWarn
    case "ERROR":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}

level := parseLogLevel(os.Getenv("LOG_LEVEL"))
handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: level,
})
```

**Log sampling for high-frequency events**:

```python
import random

class SamplingLogger:
    def __init__(self, logger, sample_rate=0.01):
        self.logger = logger
        self.sample_rate = sample_rate

    def info(self, msg, **kwargs):
        if random.random() < self.sample_rate:
            self.logger.info(msg, sample_rate=self.sample_rate, **kwargs)

# Use for high-volume endpoints
health_logger = SamplingLogger(logger, sample_rate=0.001)  # Log 0.1% of health checks
```

```go
// Go - atomic counter-based sampling
type SampledLogger struct {
    logger     *slog.Logger
    counter    atomic.Int64
    sampleRate int64  // log every N events
}

func (s *SampledLogger) Info(msg string, args ...any) {
    count := s.counter.Add(1)
    if count%s.sampleRate == 0 {
        s.logger.Info(msg, append(args,
            slog.Int64("sample_rate", s.sampleRate),
            slog.Int64("event_count", count),
        )...)
    }
}
```

**Field size limits**:

```python
def truncate_field(value, max_length=1024):
    """Truncate log field values to prevent oversized entries."""
    if isinstance(value, str) and len(value) > max_length:
        return value[:max_length] + f"...[truncated, original length: {len(value)}]"
    return value

def truncate_processor(logger, method_name, event_dict):
    """structlog processor that truncates long field values."""
    for key, value in event_dict.items():
        event_dict[key] = truncate_field(value)
    return event_dict
```

---

## 7. Language-Specific Patterns

### Python (structlog / logging)

**Recommended setup**:
```python
# Standard production configuration with structlog
import structlog
import logging
import sys

def configure_logging(log_level="INFO", json_output=True):
    """Configure structured logging for production use."""
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if json_output:
        renderer = structlog.processors.JSONRenderer()
    else:
        renderer = structlog.dev.ConsoleRenderer()

    structlog.configure(
        processors=shared_processors + [
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        processors=[
            structlog.stdlib.ProcessorFormatter.remove_processors_meta,
            renderer,
        ],
    )

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)

    root = logging.getLogger()
    root.addHandler(handler)
    root.setLevel(getattr(logging, log_level.upper()))

    # Suppress noisy third-party loggers
    logging.getLogger("urllib3").setLevel(logging.WARNING)
    logging.getLogger("botocore").setLevel(logging.WARNING)
```

**Common anti-patterns to flag**:

| Anti-Pattern | Detection | Severity |
|-------------|-----------|----------|
| `logging.basicConfig()` with no arguments | Missing level, format, handler configuration | MEDIUM |
| `logger.exception()` outside of `except` block | Generates confusing output with `NoneType` traceback | LOW |
| `logger.error(traceback.format_exc())` | Puts multiline traceback in message field; use `logger.exception()` or `exc_info=True` | LOW |
| String formatting with `%` or `.format()` in log calls | Formats string even if level is filtered; use lazy formatting | LOW |
| `print()` statements for operational logging | No level, no structure, no configuration | MEDIUM |

### Go (slog / zerolog / zap)

**Recommended setup with slog (Go 1.21+)**:
```go
package logging

import (
    "context"
    "log/slog"
    "os"
)

func NewProductionLogger(serviceName string) *slog.Logger {
    opts := &slog.HandlerOptions{
        Level:     parseLogLevel(os.Getenv("LOG_LEVEL")),
        AddSource: true,
    }
    handler := slog.NewJSONHandler(os.Stdout, opts)
    return slog.New(handler).With(
        slog.String("service", serviceName),
        slog.String("version", os.Getenv("APP_VERSION")),
    )
}

// RequestLogger creates a child logger with request-scoped fields.
func RequestLogger(logger *slog.Logger, requestID, traceID, userID string) *slog.Logger {
    return logger.With(
        slog.String("request_id", requestID),
        slog.String("trace_id", traceID),
        slog.String("user_id", userID),
    )
}
```

**Recommended setup with zerolog**:
```go
package logging

import (
    "os"
    "time"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func Init(serviceName string) {
    zerolog.TimeFieldFormat = time.RFC3339Nano
    zerolog.LevelFieldName = "level"
    zerolog.MessageFieldName = "message"
    zerolog.TimestampFieldName = "timestamp"

    log.Logger = zerolog.New(os.Stdout).
        With().
        Timestamp().
        Str("service", serviceName).
        Str("version", os.Getenv("APP_VERSION")).
        Logger()
}
```

**Common anti-patterns to flag**:

| Anti-Pattern | Detection | Severity |
|-------------|-----------|----------|
| `fmt.Println` or `log.Println` for operational logging | Unstructured, no levels | MEDIUM |
| `log.Fatal` in library code (kills the process) | Should return errors, not exit | HIGH |
| Missing `slog.Error` field when logging errors | Error context lost | MEDIUM |
| Logging inside goroutines without context propagation | Missing request_id, trace_id | MEDIUM |

### Node.js (pino / winston)

**Recommended setup with pino**:
```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: process.env.SERVICE_NAME || 'my-service',
    version: process.env.APP_VERSION || 'unknown',
    environment: process.env.NODE_ENV || 'development',
    pid: process.pid,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  serializers: {
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res,
  },
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers.cookie',
      'password',
      'token',
      'apiKey',
      'creditCard',
      'ssn',
    ],
    censor: '[REDACTED]',
  },
});

module.exports = logger;
```

**Express middleware for request logging**:
```javascript
const pinoHttp = require('pino-http');

app.use(pinoHttp({
  logger,
  genReqId: (req) => req.headers['x-request-id'] || crypto.randomUUID(),
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  customSuccessMessage: (req, res) => `${req.method} ${req.url} ${res.statusCode}`,
  customErrorMessage: (req, res, err) => `${req.method} ${req.url} failed: ${err.message}`,
}));
```

**Common anti-patterns to flag**:

| Anti-Pattern | Detection | Severity |
|-------------|-----------|----------|
| `console.log` / `console.error` in production code | No structure, no levels, no redaction | MEDIUM |
| `JSON.stringify(obj)` in log message instead of structured fields | Embedded JSON in string field | LOW |
| No error serializer configured (error details lost) | `err.message` logged but not `err.stack` | MEDIUM |
| Winston without explicit format (defaults to unstructured) | Missing `format: combine(timestamp(), json())` | MEDIUM |

### Java (slf4j / logback / Log4j2)

**Recommended logback configuration**:
```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>request_id</includeMdcKeyName>
      <includeMdcKeyName>trace_id</includeMdcKeyName>
      <includeMdcKeyName>user_id</includeMdcKeyName>
      <customFields>
        {"service":"payment-service","version":"${APP_VERSION:-unknown}"}
      </customFields>
      <timestampPattern>yyyy-MM-dd'T'HH:mm:ss.SSS'Z'</timestampPattern>
      <timeZone>UTC</timeZone>
    </encoder>
  </appender>

  <root level="${LOG_LEVEL:-INFO}">
    <appender-ref ref="JSON"/>
  </root>

  <!-- Suppress noisy libraries -->
  <logger name="org.apache.http" level="WARN"/>
  <logger name="org.hibernate" level="WARN"/>
</configuration>
```

**Using SLF4J with structured arguments**:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import static net.logstash.logback.argument.StructuredArguments.*;

public class OrderService {
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);

    public Order processOrder(String orderId) {
        logger.info("Processing order",
            keyValue("order_id", orderId),
            keyValue("user_id", getCurrentUserId()));

        try {
            Order order = orderRepository.findById(orderId);
            // ... processing ...

            logger.info("Order processed successfully",
                keyValue("order_id", orderId),
                keyValue("duration_ms", elapsed),
                keyValue("item_count", order.getItemCount()));
            return order;

        } catch (Exception e) {
            logger.error("Order processing failed",
                keyValue("order_id", orderId),
                keyValue("error_type", e.getClass().getSimpleName()),
                e);  // passing exception as last arg includes stack trace
            throw e;
        }
    }
}
```

**Common anti-patterns to flag**:

| Anti-Pattern | Detection | Severity |
|-------------|-----------|----------|
| `System.out.println` for operational logging | No levels, no structure, no MDC | MEDIUM |
| String concatenation in log calls: `logger.info("User " + userId)` | Concatenation happens even if level is filtered | LOW |
| `e.printStackTrace()` instead of `logger.error("msg", e)` | Goes to stderr, no structure, no correlation | MEDIUM |
| Missing logback/log4j2 configuration file (using defaults) | Unstructured output, no level control | MEDIUM |
| Catching and logging then re-throwing without wrapping (double logging) | Same error logged at multiple levels in the call stack | LOW |

---

## Review Procedure Summary

When evaluating logging standards:

1. **Check structured logging adoption**: Verify the project uses a structured logging library, not `print()`, `console.log()`, `fmt.Println()`, or `System.out.println()`.
2. **Verify standard field set**: Confirm timestamp, level, message, and service fields are present. Check for request_id and trace_id in HTTP handlers.
3. **Audit log level usage**: Confirm ERROR is used only for actual errors, INFO for operational events, WARN for unexpected-but-recovered conditions, DEBUG for diagnostics.
4. **Assess log content quality**: Check that log messages include sufficient context (identifiers, error details, durations) for incident investigation.
5. **Scan for sensitive data**: Use the detection patterns in Section 5 to identify PII, credentials, or tokens that may appear in log output. Verify redaction mechanisms are in place.
6. **Evaluate volume controls**: Confirm log levels are externally configurable, sampling exists for high-frequency events, and no unbounded logging occurs in loops.
7. **Check language-specific patterns**: Apply the anti-pattern checks for the project's primary language(s) from Section 7.
8. **Classify each finding** using the severity tables above, adjusting for the project's context (regulated industry increases severity for PII findings; internal tool decreases severity for volume concerns).

---

## Quick Reference: Severity Summary

| Severity | Logging Findings |
|----------|-----------------|
| CRITICAL | Passwords, tokens, API keys, credit card numbers, or SSN logged in plaintext; connection strings with credentials in logs |
| HIGH | No structured logging in a request-processing service; ERROR level for expected conditions causing alert fatigue; exceptions caught with no logging; debug level in production with no override; log storms from tight-loop logging; PII in logs in regulated systems; `log.Fatal` in Go library code |
| MEDIUM | Mixed structured/unstructured logging; missing request_id or trace_id; generic error messages without context; excessive INFO verbosity; entire request/response objects logged unfiltered; no externalized level configuration; `print`/`console.log`/`System.out.println` for operational logging |
| LOW | Inconsistent level usage across modules; minor field naming inconsistencies; string formatting style issues; commented-out log statements; missing duration for non-critical operations |
| INFO | Well-configured structured logging; consistent standard field set; proper redaction implementation; effective sampling strategy; good operational lifecycle logging |

---

## References

- [OpenTelemetry Logging Specification](https://opentelemetry.io/docs/specs/otel/logs/) -- standard for log data model and semantic conventions
- [Google SRE Book, Chapter 17: Testing for Reliability](https://sre.google/sre-book/testing-reliability/) -- operational logging expectations
- [12-Factor App: Logs](https://12factor.net/logs) -- treat logs as event streams
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) -- security-focused logging guidance
- [Python structlog documentation](https://www.structlog.org/en/stable/)
- [Go slog package documentation](https://pkg.go.dev/log/slog)
- [Pino documentation](https://getpino.io/)
- [Logback documentation](https://logback.qos.ch/documentation.html)
