# Maintainability Reference

This reference provides detailed checkpoints for maintainability concerns. It complements the inline criteria in `review-quality.md` with expanded definitions, severity guidance, and examples for each checkpoint.

The `quality-reviewer` agent uses this file as a lookup when evaluating code. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Error Handling

### 1.1 Missing Error Handlers

**Checkpoint**: Operations that can fail (file I/O, network calls, parsing, type conversions, database queries) must have explicit error handling. The absence of error handling on a fallible operation is a finding.

**Severity guidance**:

| Context | Severity |
|---|---|
| Critical path (payment, auth, data persistence) | HIGH |
| Non-critical path (logging, analytics, caching) | MEDIUM |
| Development/debug utilities | LOW |

**Example**:

Problem:
```python
def load_user_preferences(user_id):
    path = f"/data/prefs/{user_id}.json"
    with open(path) as f:           # FileNotFoundError unhandled
        data = json.load(f)         # JSONDecodeError unhandled
    return data["preferences"]      # KeyError unhandled
```

Better:
```python
def load_user_preferences(user_id):
    path = f"/data/prefs/{user_id}.json"
    try:
        with open(path) as f:
            data = json.load(f)
    except FileNotFoundError:
        logger.info("No preferences file for user %s, using defaults", user_id)
        return DEFAULT_PREFERENCES
    except json.JSONDecodeError as e:
        logger.error("Corrupt preferences file for user %s: %s", user_id, e)
        return DEFAULT_PREFERENCES

    return data.get("preferences", DEFAULT_PREFERENCES)
```

---

### 1.2 Inconsistent Error Handling Patterns

**Checkpoint**: The error handling strategy within a module or service should be uniform. Mixing error codes and exceptions, or mixing different exception hierarchies for similar operations, creates confusion and missed error paths.

**Severity**: MEDIUM

**Example**:

Problem:
```go
func (s *Service) GetUser(id string) (*User, error) {
    // Returns error -- good
    return s.db.FindUser(id)
}

func (s *Service) GetOrder(id string) *Order {
    // Returns nil on failure -- inconsistent with GetUser
    order, err := s.db.FindOrder(id)
    if err != nil {
        log.Printf("order not found: %v", err)
        return nil
    }
    return order
}
```

Better:
```go
func (s *Service) GetUser(id string) (*User, error) {
    return s.db.FindUser(id)
}

func (s *Service) GetOrder(id string) (*Order, error) {
    return s.db.FindOrder(id)
}
```

---

### 1.3 Resource Cleanup

**Checkpoint**: Resources (file handles, database connections, network sockets, locks, temporary files) that are opened must be guaranteed to close, even when errors occur. Look for resource acquisition without corresponding cleanup in error paths.

**Severity**: HIGH for connections and locks in server code. MEDIUM for file handles and temporary resources.

**Example**:

Problem:
```python
def export_report(query):
    conn = db.connect()
    cursor = conn.cursor()
    cursor.execute(query)
    rows = cursor.fetchall()
    output = open("/tmp/report.csv", "w")
    writer = csv.writer(output)
    writer.writerows(rows)
    output.close()     # never reached if writerows() fails
    conn.close()       # never reached if anything above fails
```

Better:
```python
def export_report(query):
    with db.connect() as conn:
        cursor = conn.cursor()
        cursor.execute(query)
        rows = cursor.fetchall()

    with open("/tmp/report.csv", "w") as output:
        writer = csv.writer(output)
        writer.writerows(rows)
```

---

## 2. Readability

### 2.1 Complex Expressions Without Decomposition

**Checkpoint**: Expressions that combine multiple operations, conditions, or transformations in a single line should be broken into named intermediate variables or extracted into helper functions. If a reader must mentally simulate the expression to understand it, it is too complex.

**Severity**: MEDIUM

**Example**:

Problem:
```python
eligible = [u for u in users if u.age >= 18 and u.active and not u.banned and
            (u.role in admin_roles or (u.verified and u.score > threshold))]
```

Better:
```python
def is_eligible(user, admin_roles, threshold):
    if user.age < 18 or not user.active or user.banned:
        return False
    if user.role in admin_roles:
        return True
    return user.verified and user.score > threshold

eligible = [u for u in users if is_eligible(u, admin_roles, threshold)]
```

---

### 2.2 Missing Documentation on Complex Logic

**Checkpoint**: Code that implements non-obvious algorithms, business rules, workarounds, or domain-specific logic should have explanatory comments or docstrings. The comment should explain *why*, not *what* (the code shows *what*).

**Severity guidance**:

| Context | Severity |
|---|---|
| Complex algorithm with no explanation | MEDIUM |
| Business rule that is not self-evident from the code | MEDIUM |
| Workaround for a known bug or limitation | MEDIUM (should reference the bug) |
| Public API with no docstring | MEDIUM |
| Internal helper with self-evident behavior | INFO (no comment needed) |

**Example**:

Problem:
```python
def calculate_score(events):
    weights = [0.5 ** (i / 7) for i in range(len(events))]
    return sum(e.value * w for e, w in zip(events, weights))
```

Better:
```python
def calculate_score(events):
    """Calculate time-decayed engagement score.

    Uses exponential decay with a half-life of 7 days. Recent events
    contribute more to the score than older ones. Events are assumed
    to be sorted newest-first.

    Reference: https://internal.wiki/scoring-algorithm
    """
    HALF_LIFE_DAYS = 7
    weights = [0.5 ** (i / HALF_LIFE_DAYS) for i in range(len(events))]
    return sum(e.value * w for e, w in zip(events, weights))
```

---

### 2.3 Misleading or Stale Comments

**Checkpoint**: Comments that contradict the code they describe are worse than no comments. Stale comments from prior implementations mislead readers. TODO/FIXME/HACK comments without associated tracking indicate unfinished work.

**Severity guidance**:

| Type | Severity |
|---|---|
| Comment contradicts the code | HIGH |
| Comment describes removed/changed functionality | MEDIUM |
| TODO/FIXME/HACK without tracking reference | LOW |
| Obvious comment that restates the code | INFO (unnecessary noise) |

**Example**:

Problem:
```python
# Retry up to 3 times with exponential backoff
for attempt in range(5):  # <-- actually 5, not 3
    try:
        return fetch(url)
    except Timeout:
        time.sleep(attempt * 2)  # <-- linear, not exponential
```

Better:
```python
MAX_RETRIES = 5
BACKOFF_MULTIPLIER = 2

# Retry with linear backoff: 0s, 2s, 4s, 6s, 8s
for attempt in range(MAX_RETRIES):
    try:
        return fetch(url)
    except Timeout:
        time.sleep(attempt * BACKOFF_MULTIPLIER)
```

---

## 3. Configuration

### 3.1 Hardcoded Values

**Checkpoint**: Values that are likely to change between environments, deployments, or over time should be externalized to configuration. This includes URLs, ports, timeouts, retry counts, thresholds, feature flags, and queue/topic names. Secrets are excluded from this checkpoint (those are CRITICAL security findings).

**Severity**: MEDIUM for non-secret values. Escalate to HIGH if the hardcoded value is environment-specific (e.g., a production URL in source code).

**Example**:

Problem:
```python
def fetch_prices():
    response = requests.get(
        "https://api.pricing.internal.example.com/v2/prices",
        timeout=30,
    )
    if len(response.json()) > 1000:
        logger.warning("Too many prices")
    return response.json()
```

Better:
```python
def fetch_prices(config: AppConfig):
    response = requests.get(
        config.pricing_api_url,
        timeout=config.pricing_timeout_seconds,
    )
    if len(response.json()) > config.max_price_count:
        logger.warning("Price count %d exceeds limit %d",
                       len(response.json()), config.max_price_count)
    return response.json()
```

---

### 3.2 Environment-Specific Code

**Checkpoint**: Code that behaves differently based on the environment should use configuration or feature flags, not inline conditionals that reference environment names. Hardcoded environment checks (`if env == "production"`) scatter environment awareness throughout the codebase.

**Severity**: HIGH. Environment-specific branches are a frequent source of bugs when a new environment is added or when the conditional is missed during refactoring.

**Example**:

Problem:
```python
def get_database_url():
    if os.environ.get("ENV") == "production":
        return "postgresql://prod-db.internal:5432/app"
    elif os.environ.get("ENV") == "staging":
        return "postgresql://staging-db.internal:5432/app"
    else:
        return "postgresql://localhost:5432/app"
```

Better:
```python
def get_database_url():
    url = os.environ.get("DATABASE_URL")
    if not url:
        raise ConfigError("DATABASE_URL environment variable is required")
    return url
```

---

### 3.3 Missing Configuration Validation

**Checkpoint**: Configuration values loaded from environment variables, files, or external sources should be validated at startup. Missing or invalid configuration should fail fast with a clear error message, not cause cryptic failures later at runtime.

**Severity**: MEDIUM for missing validation. HIGH if invalid configuration could cause data loss or security issues.

**Example**:

Problem:
```python
class AppConfig:
    def __init__(self):
        self.port = int(os.environ.get("PORT", "8080"))
        self.db_url = os.environ["DATABASE_URL"]  # KeyError if missing
        self.workers = os.environ.get("WORKERS")   # stays as string, no int conversion
        self.timeout = os.environ.get("TIMEOUT")   # None if missing, used later as number
```

Better:
```python
class AppConfig:
    def __init__(self):
        self.port = self._require_int("PORT", default=8080, min_val=1, max_val=65535)
        self.db_url = self._require("DATABASE_URL")
        self.workers = self._require_int("WORKERS", default=4, min_val=1)
        self.timeout = self._require_float("TIMEOUT", default=30.0, min_val=0.1)

    def _require(self, key, default=None):
        value = os.environ.get(key, default)
        if value is None:
            raise ConfigError(f"Required environment variable {key} is not set")
        return value

    def _require_int(self, key, default=None, min_val=None, max_val=None):
        raw = self._require(key, default=str(default) if default is not None else None)
        try:
            value = int(raw)
        except (ValueError, TypeError):
            raise ConfigError(f"{key} must be an integer, got: {raw!r}")
        if min_val is not None and value < min_val:
            raise ConfigError(f"{key} must be >= {min_val}, got: {value}")
        if max_val is not None and value > max_val:
            raise ConfigError(f"{key} must be <= {max_val}, got: {value}")
        return value
```

---

## 4. Structure

### 4.1 Inconsistent Project Organization

**Checkpoint**: Files and directories should follow a consistent organizational scheme. Similar components should be located in similar places. The structure should reflect the architecture (by layer, by feature, or by domain) and that choice should be applied uniformly.

**Severity**: LOW for minor inconsistencies. MEDIUM when the inconsistency makes it difficult to find code or understand the architecture.

**Example**:

Problem:
```
src/
  controllers/
    user_controller.py
    order_controller.py
  services/
    user_service.py
  models/
    user.py
    order.py
  helpers/
    order_helper.py          # should this be in services/?
  utils/
    user_utils.py            # overlaps with helpers/
  order_business_logic.py    # why is this at the root?
```

Better:
```
src/
  controllers/
    user_controller.py
    order_controller.py
  services/
    user_service.py
    order_service.py          # consistent with user_service.py
  models/
    user.py
    order.py
```

---

### 4.2 Tangled Dependencies

**Checkpoint**: The dependency graph between modules should be acyclic and follow a clear direction (e.g., controllers depend on services, services depend on models, not the reverse). Cross-layer dependencies and unexpected dependency directions are findings.

**Severity guidance**:

| Type | Severity |
|---|---|
| Circular dependency between modules | HIGH |
| Lower layer importing from higher layer | HIGH |
| Cross-feature dependency that should go through a shared interface | MEDIUM |
| Tight coupling to concrete implementations where an interface would be appropriate | MEDIUM |

**Example**:

Problem:
```python
# models/user.py
from controllers.user_controller import format_user_response  # model imports controller

# services/order_service.py
from services.user_service import UserService       # OK
from services.payment_service import PaymentService  # OK
from controllers.order_controller import validate_order_params  # service imports controller
```

Better:
```python
# models/user.py -- depends on nothing
class User:
    def to_dict(self): ...

# services/order_service.py -- depends on models and other services
from models.order import Order
from services.user_service import UserService
from services.payment_service import PaymentService

# controllers/order_controller.py -- depends on services
from services.order_service import OrderService
```

---

### 4.3 Unclear Module Boundaries

**Checkpoint**: Each module or package should have a clear, defined responsibility. Its public interface (exports) should be intentional. Internal implementation details should not leak across module boundaries.

**Severity**: MEDIUM when boundaries are unclear. HIGH when internal details are accessed from outside the module.

**Example**:

Problem:
```python
# billing/internal/tax_calculator.py
def _apply_regional_tax(amount, region):
    """Internal implementation detail."""
    ...

# shipping/cost_calculator.py
from billing.internal.tax_calculator import _apply_regional_tax  # reaching into internals
```

Better:
```python
# billing/__init__.py (or billing/api.py)
def calculate_tax(amount, region):
    """Public interface for tax calculation."""
    return _internal.apply_regional_tax(amount, region)

# shipping/cost_calculator.py
from billing import calculate_tax  # uses public interface
```

---

## 5. API Stability

### 5.1 Breaking Changes Without Version Bump

**Checkpoint**: Changes to public APIs (function signatures, return types, endpoint contracts, exported symbols) that break existing callers must be accompanied by a version bump. In libraries, this means a major version increment. In services, this means a new API version.

**Severity**: HIGH for libraries consumed by other teams or external users. MEDIUM for internal APIs with known consumers.

**Signals to check**:
- Removed or renamed public functions, classes, or methods
- Changed parameter names, types, or order in public functions
- Changed return type or structure
- Removed fields from response objects
- Changed HTTP method, URL path, or status codes on API endpoints
- Changed configuration file schema

**Example**:

Problem:
```python
# v1.2.3 -- public API
def create_user(name: str, email: str) -> User:
    ...

# Changed to (in same version):
def create_user(email: str, name: str, role: str = "user") -> dict:
    # parameter order changed, return type changed, no version bump
    ...
```

Better:
```python
# v2.0.0 -- major version bump signals breaking change
def create_user(email: str, name: str, role: str = "user") -> dict:
    ...

# Or, preserve backward compatibility in v1.x:
def create_user(name: str, email: str, role: str = "user") -> User:
    ...
```

---

### 5.2 Missing Deprecation Path

**Checkpoint**: Public APIs that are being replaced should go through a deprecation period before removal. The old API should emit deprecation warnings pointing to the replacement. Removing an API without deprecation forces consumers into emergency migrations.

**Severity**: HIGH for external/public APIs. MEDIUM for internal APIs.

**Example**:

Problem:
```python
# Old API deleted in this commit, no deprecation:
# def get_user_by_email(email): ...   <-- just removed

# New API:
def find_user(email=None, username=None):
    ...
```

Better:
```python
import warnings

def get_user_by_email(email):
    """Deprecated: Use find_user(email=...) instead. Will be removed in v3.0."""
    warnings.warn(
        "get_user_by_email() is deprecated, use find_user(email=...) instead. "
        "Scheduled for removal in v3.0.",
        DeprecationWarning,
        stacklevel=2,
    )
    return find_user(email=email)

def find_user(email=None, username=None):
    ...
```

---

### 5.3 Semantic Versioning Compliance

**Checkpoint**: Version numbers should follow semantic versioning (or the project's declared versioning scheme). Bug fixes increment patch, backward-compatible additions increment minor, breaking changes increment major. Mismatched version bumps erode consumer trust.

**Severity**: MEDIUM when the version increment does not match the change type. LOW for pre-1.0 projects where the contract is explicitly unstable.

**Signals to check**:
- Breaking change with only a minor or patch version bump
- New feature addition with only a patch version bump
- Version string not updated when the changelog or release notes describe changes

---

## Quick-Reference Summary

| Area | Checkpoint | Severity |
|---|---|---|
| Error Handling | Missing handlers on fallible operations | HIGH (critical path) / MEDIUM |
| Error Handling | Inconsistent error handling patterns | MEDIUM |
| Error Handling | Resource cleanup not guaranteed on error | HIGH (connections/locks) / MEDIUM |
| Readability | Complex expressions without decomposition | MEDIUM |
| Readability | Missing documentation on complex logic | MEDIUM |
| Readability | Comments contradict code | HIGH |
| Readability | Stale comments from prior implementation | MEDIUM |
| Readability | TODO/FIXME without tracking | LOW |
| Configuration | Hardcoded non-secret values | MEDIUM |
| Configuration | Environment-specific code (inline conditionals) | HIGH |
| Configuration | Missing configuration validation | MEDIUM-HIGH |
| Structure | Inconsistent project organization | LOW-MEDIUM |
| Structure | Tangled/circular dependencies | HIGH |
| Structure | Internal details leaking across module boundaries | MEDIUM-HIGH |
| API Stability | Breaking changes without version bump | HIGH |
| API Stability | Removed API without deprecation path | HIGH (public) / MEDIUM (internal) |
| API Stability | Version increment mismatches change type | MEDIUM |
