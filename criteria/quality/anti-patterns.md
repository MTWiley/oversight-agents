# Anti-Patterns Reference

This reference provides detailed detection criteria for common anti-patterns. It complements the inline criteria in `review-quality.md` with expanded definitions, harm analysis, detection signals, and recommended alternatives.

The `quality-reviewer` agent uses this file as a lookup when evaluating code. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Structural Anti-Patterns

### God Object

**Definition**: A class or module that concentrates too many responsibilities, becoming the central hub that most other components depend on. It knows about or controls too much of the system.

**Why it is harmful**: God objects resist change because any modification risks breaking unrelated functionality. They cannot be tested in isolation, they create merge conflicts when multiple developers work on different features, and they make it impossible to deploy or scale components independently.

**Detection signals**:
- Class exceeds 500 lines or 20 methods
- More than 10 direct dependency imports
- Most other modules import this class
- Vague naming: "AppManager", "MainController", "SystemHelper"
- Methods that span unrelated domains (authentication, logging, persistence, UI in one class)
- The class is modified in nearly every pull request regardless of what feature is being built

**Severity**: HIGH

**Recommended alternative**: Decompose into focused classes, each owning a single responsibility. Apply the Single Responsibility Principle: each class should have one reason to change. Use composition rather than aggregation -- the former god object can become a thin coordinator that delegates to specialized collaborators.

**Example**:

Anti-pattern:
```python
class ApplicationManager:
    def authenticate_user(self, credentials): ...
    def authorize_action(self, user, action): ...
    def fetch_data(self, query): ...
    def transform_data(self, raw_data): ...
    def render_template(self, template, data): ...
    def send_email(self, recipient, body): ...
    def write_log(self, level, message): ...
    def update_cache(self, key, value): ...
    def run_migration(self, version): ...
```

Alternative:
```python
class Authenticator:
    def authenticate(self, credentials): ...
    def authorize(self, user, action): ...

class DataService:
    def fetch(self, query): ...
    def transform(self, raw_data): ...

class Renderer:
    def render(self, template, data): ...

class Notifier:
    def send_email(self, recipient, body): ...
```

---

### Anemic Domain Model

**Definition**: Domain objects that contain only data (getters/setters) with all behavior living in separate "service" or "manager" classes. The domain model becomes a glorified data transfer structure with no encapsulation of business rules.

**Why it is harmful**: Business logic scatters across service classes, making it hard to find the authoritative implementation of a rule. Invariants that should be enforced by the domain object must instead be enforced by every caller. Duplicate logic emerges because multiple services implement the same validation or calculation independently.

**Detection signals**:
- Classes with only fields, getters, and setters (or `@dataclass`/`struct` equivalents) but no behavior
- Separate "Service" classes that take a domain object and perform all operations on its data
- Validation logic living outside the object being validated
- Business rules duplicated across multiple service methods

**Severity**: MEDIUM. Code works correctly if all callers are disciplined, but the architecture is fragile and invites divergence.

**Recommended alternative**: Push behavior into domain objects. If an `Order` has rules about when it can be cancelled, that logic belongs on `Order.cancel()`, not on `OrderService.cancel_order(order)`. Services should orchestrate (coordinate transactions, call external systems) rather than implement domain logic.

**Example**:

Anti-pattern:
```python
@dataclass
class Order:
    items: list
    status: str
    total: float

class OrderService:
    def cancel(self, order: Order):
        if order.status not in ("pending", "confirmed"):
            raise ValueError("Cannot cancel")
        order.status = "cancelled"
        order.total = 0
```

Alternative:
```python
class Order:
    def __init__(self, items, status="pending"):
        self.items = items
        self.status = status
        self.total = self._compute_total()

    def cancel(self):
        if self.status not in ("pending", "confirmed"):
            raise ValueError(f"Cannot cancel order in {self.status} state")
        self.status = "cancelled"
        self.total = 0

    def _compute_total(self):
        return sum(item.price * item.quantity for item in self.items)
```

---

### Big Ball of Mud

**Definition**: A system with no discernible architecture -- components reference each other freely, layers are not enforced, and there is no clear separation of concerns. Everything depends on everything.

**Why it is harmful**: Changes in any part of the system risk cascading failures. Onboarding new developers is expensive. Testing requires the entire system to be available. The system cannot evolve incrementally; improvements require large, risky rewrites.

**Detection signals**:
- Circular dependencies between packages or modules at every level
- No clear directory structure corresponding to architectural layers or domains
- Import statements that reach across boundaries (e.g., UI code importing database internals)
- Inability to describe the architecture in a sentence or draw a dependency diagram without cycles
- Test files that must import from many unrelated modules to set up fixtures

**Severity**: HIGH when detected in new code being introduced. INFO when observing existing structure (recommending incremental improvement rather than rewrite).

**Recommended alternative**: Introduce boundaries incrementally. Start with dependency direction: define which modules are allowed to depend on which. Use linter rules or import restrictions to enforce direction. Gradually extract shared abstractions into a core layer that depends on nothing else.

---

## 2. Behavioral Anti-Patterns

### Premature Optimization

**Definition**: Introducing complex, hard-to-read code in pursuit of performance gains that have not been measured or that occur in non-critical paths. Sacrificing clarity for speed without evidence that speed is needed.

**Why it is harmful**: Complex optimized code is harder to understand, harder to modify, and more likely to contain bugs. The performance gain is often negligible or occurs in a code path that accounts for a tiny fraction of execution time. Meanwhile, the maintenance cost is paid on every future read and modification.

**Detection signals**:
- Bitwise operations, manual memory management, or custom data structures where standard library equivalents exist
- Comments like "this is faster" without benchmark data
- Optimization that sacrifices error handling, input validation, or correctness
- Caching added before any evidence of repeated expensive computation
- Pre-allocation of large buffers "just in case"

**Severity**: MEDIUM for readability impact. HIGH if the optimization sacrifices correctness or error handling.

**Recommended alternative**: Write clear, correct code first. Profile before optimizing. When optimization is needed, document the measured bottleneck and benchmark results. Keep the readable version available as a comment or in commit history.

**Example**:

Anti-pattern:
```python
# "Optimized" string building
def build_message(parts):
    buf = bytearray(4096)  # pre-allocated, "faster"
    offset = 0
    for p in parts:
        encoded = p.encode('utf-8')
        buf[offset:offset+len(encoded)] = encoded
        offset += len(encoded)
    return buf[:offset].decode('utf-8')
```

Alternative:
```python
def build_message(parts):
    return "".join(parts)
```

---

### Stringly-Typed Code

**Definition**: Using plain strings to represent values that have a defined, finite set of valid states or that carry structured data. Strings provide no compile-time or static-analysis safety, enable typos, and resist refactoring tools.

**Why it is harmful**: Typos in string values produce silent bugs. Refactoring is unsafe because search-and-replace cannot distinguish between a status string and an unrelated string with the same value. Structured data encoded as delimited strings requires custom parsing logic that duplicates format knowledge across the codebase.

**Detection signals**:
- String comparisons against a fixed set of values: `if status == "actve"` (spot the typo)
- String concatenation to build structured data: `key = f"{user_id}:{session_id}:{timestamp}"`
- Dictionary keys that represent a finite domain: `config["mode"]` compared against `"fast"`, `"slow"`, `"balanced"`
- Switch/match on string values that represent types or categories

**Severity**: MEDIUM for fixed-set values that should be enums. HIGH for structured data passed as concatenated strings.

**Recommended alternative**: Use enums, constants, or dedicated types for values with a fixed domain. Use dataclasses, structs, typed dictionaries, or dedicated types for structured data.

**Example**:

Anti-pattern:
```python
def set_priority(task, priority):
    if priority not in ("low", "medium", "high", "critical"):
        raise ValueError("Invalid priority")
    task["priority"] = priority

# Caller can easily pass "Low" or "HIGH" (wrong case) or "med"
set_priority(task, "hgih")  # typo, caught only at runtime
```

Alternative:
```python
from enum import Enum

class Priority(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

def set_priority(task, priority: Priority):
    task.priority = priority

set_priority(task, Priority.HIGH)  # type-safe, IDE-assisted
```

---

### Callback Hell / Pyramid of Doom

**Definition**: Deeply nested callbacks where each asynchronous operation's result handler contains the next operation, creating a pyramid of indentation that is hard to read, hard to debug, and hard to add error handling to.

**Why it is harmful**: Error handling must be duplicated at each nesting level. The flow of execution is inverted -- you read the code from inside out rather than top to bottom. Adding a step in the middle requires restructuring the entire chain. Debugging stack traces become opaque.

**Detection signals**:
- Callback nesting deeper than 3 levels
- Async operations where `async`/`await`, promises, or reactive patterns are available in the language
- Error handling duplicated or missing at inner nesting levels
- Indentation width exceeding half the screen/editor width

**Severity**: MEDIUM when async/await alternatives exist in the language runtime.

**Recommended alternative**: Use `async`/`await`, promises/futures, or reactive streams. If the language does not support these, extract each step into a named function to flatten the nesting.

**Example**:

Anti-pattern:
```javascript
getUser(userId, function(err, user) {
    if (err) return handleError(err);
    getOrders(user.id, function(err, orders) {
        if (err) return handleError(err);
        getPayments(orders[0].id, function(err, payments) {
            if (err) return handleError(err);
            generateInvoice(payments, function(err, invoice) {
                if (err) return handleError(err);
                sendEmail(user.email, invoice, function(err) {
                    if (err) return handleError(err);
                    console.log("Done");
                });
            });
        });
    });
});
```

Alternative:
```javascript
async function processInvoice(userId) {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const payments = await getPayments(orders[0].id);
    const invoice = await generateInvoice(payments);
    await sendEmail(user.email, invoice);
}
```

---

### Golden Hammer

**Definition**: Applying a familiar technology, pattern, or tool to every problem regardless of fit. "When all you have is a hammer, everything looks like a nail."

**Why it is harmful**: Forces inappropriate abstractions, adds unnecessary complexity, and ignores better-suited alternatives. Often produces brittle solutions that fight the tool rather than leveraging the problem domain's natural structure.

**Detection signals**:
- A single pattern (e.g., Observer, Factory, microservices) applied uniformly across the entire codebase regardless of context
- Libraries pulled in for trivial tasks (e.g., a full ORM for a script that runs one query)
- Architecture choices that match the team's prior project rather than the current project's requirements
- Overly abstract layers wrapping simple operations

**Severity**: MEDIUM. The code works but carries unnecessary complexity and maintenance cost.

**Recommended alternative**: Choose tools and patterns based on the problem at hand. Apply the simplest approach that meets current and reasonably foreseeable requirements. Document why a particular pattern was chosen when it is non-obvious.

---

## 3. Coupling Anti-Patterns

### Inappropriate Intimacy

**Definition**: Two classes or modules that access each other's internal/private state, bypassing public interfaces. They are so tightly intertwined that neither can change independently.

**Why it is harmful**: Changes to internal data structures in one class break the other. Encapsulation is violated, removing the ability to enforce invariants. Testing either class in isolation becomes impossible.

**Detection signals**:
- Direct access to private/protected fields from outside the class (`obj._internal_field`)
- Reaching into another module's internal data structures
- Two classes that always change together
- Friend classes or package-private access used as a workaround for design problems
- Law of Demeter violations: `a.b.c.d.doSomething()`

**Severity**: HIGH

**Recommended alternative**: Expose behavior through well-defined public interfaces. If class A needs information from class B's internals, add a method on B that provides the information without exposing the structure. Consider whether the classes should be merged or whether a mediator is needed.

**Example**:

Anti-pattern:
```python
class OrderProcessor:
    def calculate_shipping(self, order):
        # Reaching into customer's internals
        if order.customer._address_cache["primary"]["region"] == "domestic":
            return order._line_items_internal[0]._weight_kg * 2.5
```

Alternative:
```python
class Customer:
    def is_domestic(self) -> bool:
        return self.primary_address.region == "domestic"

class Order:
    def total_weight_kg(self) -> float:
        return sum(item.weight_kg for item in self.line_items)

class OrderProcessor:
    def calculate_shipping(self, order):
        if order.customer.is_domestic():
            return order.total_weight_kg() * 2.5
```

---

### Shotgun Surgery

**Definition**: A single logical change requires modifications scattered across many unrelated files. The knowledge about a concept is spread so thin that even small changes require a broad search.

**Why it is harmful**: Changes are error-prone because developers must find and update every affected location. Missed locations become bugs. Code reviews are difficult because the reviewer must understand the relationship between many small changes in different files.

**Detection signals**:
- Adding a new enum value, field, or variant requires changes in 5+ files
- Adding a new feature type requires updating switch statements in multiple locations
- A new configuration option must be threaded through many layers manually
- Git history shows that certain groups of files always change together

**Severity**: MEDIUM for 3-4 affected files. HIGH for 5+ affected files.

**Recommended alternative**: Consolidate the scattered knowledge. Use polymorphism or dispatch tables to localize type-specific behavior. Use configuration-driven approaches where adding a new variant means adding data, not modifying code in many places.

**Example**:

Anti-pattern (adding a new report type requires changes in 6 files):
```
report_types.py     - add "quarterly" to VALID_TYPES list
report_factory.py   - add elif for "quarterly"
report_validator.py - add elif for "quarterly" validation rules
report_renderer.py  - add elif for "quarterly" rendering
report_exporter.py  - add elif for "quarterly" export format
menu.py             - add "quarterly" to dropdown options
```

Alternative (adding a new report type means adding one file):
```python
# reports/quarterly.py
class QuarterlyReport(BaseReport):
    type_name = "quarterly"
    def validate(self): ...
    def render(self): ...
    def export(self): ...

# Auto-discovered by registry:
# BaseReport subclasses are found automatically via __init_subclass__
```

---

### Feature Envy

**Definition**: A method that uses another class's data more than its own, suggesting the logic belongs on that other class.

**Why it is harmful**: The behavior is in the wrong place. The envied class cannot enforce its invariants because the logic operating on its data lives elsewhere. Changes to the envied class's data structure break the envious method.

**Detection signals**:
- A method that takes an object and accesses 3+ of its fields to compute a result
- More references to another object's attributes than to `self`/`this`
- The method would not need any data from its own class to function

**Severity**: MEDIUM

**Recommended alternative**: Move the method to the class whose data it primarily uses. If the method uses data from multiple classes, consider extracting the computation into the class that owns the most accessed data, or introduce a new class that represents the combined concept.

---

### Circular Dependencies

**Definition**: Module A depends on module B, and module B depends on module A (directly or through a chain). The dependency graph contains cycles.

**Why it is harmful**: Circular dependencies make it impossible to understand, test, or deploy modules independently. Import order becomes fragile, causing initialization errors. Refactoring is dangerous because the coupling is bidirectional.

**Detection signals**:
- Import errors that depend on module loading order
- Two modules that cannot be used without each other
- Dependency graphs (from tooling) showing cycles
- Deferred imports (`import inside function body`) used as a workaround for circular imports

**Severity**: HIGH

**Recommended alternative**: Break the cycle by extracting the shared concept into a third module that both depend on. Alternatively, use dependency inversion: define an interface in the lower-level module and have the higher-level module implement it. Events or callbacks can also decouple bidirectional communication.

**Example**:

Anti-pattern:
```python
# auth.py
from permissions import check_permission  # depends on permissions
def login(user): ...

# permissions.py
from auth import get_current_user  # depends on auth -- circular
def check_permission(action): ...
```

Alternative:
```python
# user_context.py (new, shared module)
def get_current_user(): ...

# auth.py
from permissions import check_permission
def login(user): ...

# permissions.py
from user_context import get_current_user  # no cycle
def check_permission(action): ...
```

---

## 4. State Anti-Patterns

### Mutable Global State

**Definition**: Global variables or module-level state that is modified during program execution by multiple functions. The program's behavior depends on the order of operations and the history of calls.

**Why it is harmful**: Functions that read or write global state have hidden inputs and outputs not reflected in their signatures. Testing requires resetting global state between tests, making tests order-dependent. Concurrency becomes dangerous because multiple threads can race on the shared state.

**Detection signals**:
- Module-level variables modified by functions: `global counter` or direct module attribute mutation
- Functions whose output changes between identical calls (not due to I/O or time)
- Tests that fail when run in a different order or in parallel
- Functions that require "initialization" before they can be called

**Severity**: HIGH when multiple functions modify the state. MEDIUM for module-level state that creates hidden coupling but is only written once (configuration loaded at startup).

**Recommended alternative**: Pass state explicitly through function parameters and return values. For configuration, load once at startup and pass as an immutable object. For cross-cutting concerns like logging, use established patterns (dependency injection, context objects).

**Example**:

Anti-pattern:
```python
# shared state
request_count = 0
last_error = None

def handle_request(req):
    global request_count, last_error
    request_count += 1
    try:
        return process(req)
    except Exception as e:
        last_error = e
        raise
```

Alternative:
```python
@dataclass
class RequestMetrics:
    count: int = 0
    last_error: Optional[Exception] = None

    def record_request(self):
        self.count += 1

    def record_error(self, error: Exception):
        self.last_error = error

def handle_request(req, metrics: RequestMetrics):
    metrics.record_request()
    try:
        return process(req)
    except Exception as e:
        metrics.record_error(e)
        raise
```

---

### Singleton Abuse

**Definition**: Using the Singleton pattern as a convenient way to access global state rather than as a genuine constraint on instance count. The singleton becomes a hidden dependency injected via global access rather than explicit parameters.

**Why it is harmful**: Singletons create hidden coupling: any code can depend on the singleton without declaring it as a parameter. Testing requires monkey-patching or resetting the singleton between tests. Multiple singletons accessing each other create an invisible dependency graph.

**Detection signals**:
- `SomeClass.get_instance()` or `SomeClass.shared` called from many unrelated modules
- Singletons that hold mutable state modified during normal operation (not just configuration)
- Difficulty testing classes in isolation because they internally reach for a singleton
- More than 3-4 singletons in the same codebase

**Severity**: HIGH when the singleton holds mutable state. MEDIUM for read-only singletons used for configuration.

**Recommended alternative**: Use dependency injection. Pass the dependency explicitly through constructors or function parameters. Framework-level DI containers can manage the singleton lifecycle without hiding the dependency.

---

### Hidden Side Effects

**Definition**: Functions that modify state (database writes, file system changes, network calls, global variable mutations) without their name, signature, or documentation indicating they do so. The caller believes they are calling a pure computation or query.

**Why it is harmful**: Callers cannot reason about what a function does without reading its implementation. Side effects in unexpected places create bugs that are hard to reproduce and diagnose. Functions become unsafe to call in tests, background jobs, or dry-run modes.

**Detection signals**:
- Functions named `get_*`, `find_*`, `calculate_*`, `is_*`, `has_*` that also modify state
- Read-only-looking methods that trigger writes, deletions, or external calls
- Functions whose behavior changes the system state but whose return value is the primary documented output
- Tests that fail due to unexpected database writes or network calls from "read" functions

**Severity**: HIGH. This is one of the most harmful patterns because it violates fundamental assumptions about function behavior.

**Recommended alternative**: Separate queries from commands. Functions that return data should not modify state. Functions that modify state should make that clear in their name (`save_`, `update_`, `delete_`, `send_`). When a read and write must occur together, make the name reflect both actions.

**Example**:

Anti-pattern:
```python
def get_user_count():
    """Looks like a read, but also cleans up expired sessions."""
    db.sessions.delete_many({"expired": True})
    return db.users.count()
```

Alternative:
```python
def purge_expired_sessions():
    db.sessions.delete_many({"expired": True})

def get_user_count():
    return db.users.count()
```

---

## 5. Error Handling Anti-Patterns

### Swallowed Exceptions

**Definition**: Catching exceptions and doing nothing (or logging and continuing) without addressing the error condition. The program proceeds as if nothing went wrong, leading to corrupted state or silent data loss.

**Why it is harmful**: The failure that caused the exception is hidden. Downstream code operates on invalid or incomplete data. Debugging becomes extremely difficult because the root cause is silently suppressed. In production, data corruption accumulates undetected.

**Detection signals**:
- Empty `catch`/`except` blocks: `except Exception: pass`
- Catch blocks that only log and continue without re-raising or returning an error
- Broad catch covering an entire function body where specific exceptions are expected
- Comments like `# TODO: handle this properly` in catch blocks

**Severity**: HIGH. This is especially dangerous in data pipelines, financial transactions, and state-modifying operations.

**Recommended alternative**: Handle the specific exceptions you expect. Let unexpected exceptions propagate. If you must catch broadly for resilience (e.g., a worker loop), log with full context and report the error through monitoring. Never silently suppress.

**Example**:

Anti-pattern:
```python
def save_user(user):
    try:
        db.users.insert(user.to_dict())
        audit_log.record("user_created", user.id)
        send_welcome_email(user.email)
    except Exception:
        pass  # caller thinks save succeeded
```

Alternative:
```python
def save_user(user):
    db.users.insert(user.to_dict())  # let DB errors propagate

    try:
        audit_log.record("user_created", user.id)
    except AuditLogError as e:
        logger.warning("Audit log failed for user %s: %s", user.id, e)
        # Audit failure is non-critical; user was saved

    try:
        send_welcome_email(user.email)
    except EmailError as e:
        logger.warning("Welcome email failed for %s: %s", user.email, e)
        # Queue for retry rather than silently dropping
        email_retry_queue.enqueue(user.email, "welcome")
```

---

### Error Code / Exception Mixing

**Definition**: Using both error codes (return values) and exceptions to signal errors within the same module or codebase, forcing callers to handle two different error propagation mechanisms.

**Why it is harmful**: Callers must check both the return value and handle exceptions, doubling the error handling surface. It is easy to forget one mechanism, leading to unhandled errors. The inconsistency makes the codebase harder to learn and review.

**Detection signals**:
- Some functions return `(result, error)` tuples while others raise exceptions
- Functions that return `None` or `-1` to signal failure in the same codebase where other functions raise
- Mixed patterns within the same class or module
- Callers that check return values but do not wrap calls in try/except (or vice versa)

**Severity**: MEDIUM

**Recommended alternative**: Choose one error handling strategy per codebase (or per language convention) and apply it consistently. In Python, Java, JavaScript, C#: use exceptions. In Go: use error returns. In Rust: use `Result<T, E>`. When wrapping a library that uses a different convention, adapt at the boundary.

**Example**:

Anti-pattern:
```python
class UserService:
    def create_user(self, data):
        """Returns None on failure."""
        if not data.get("email"):
            return None  # error code style
        db.users.insert(data)
        return User(data)

    def delete_user(self, user_id):
        """Raises on failure."""
        user = db.users.find(user_id)
        if not user:
            raise UserNotFoundError(user_id)  # exception style
        db.users.delete(user_id)
```

Alternative:
```python
class UserService:
    def create_user(self, data):
        if not data.get("email"):
            raise ValidationError("email is required")
        db.users.insert(data)
        return User(data)

    def delete_user(self, user_id):
        user = db.users.find(user_id)
        if not user:
            raise UserNotFoundError(user_id)
        db.users.delete(user_id)
```

---

### Overly Broad Catch

**Definition**: Catching a base exception type (e.g., `Exception`, `Throwable`, `Error`) when only specific exceptions are expected. This catches errors that should propagate, including programming bugs.

**Why it is harmful**: Catches and handles `KeyboardInterrupt`, `SystemExit`, `OutOfMemoryError`, `NullPointerException`, or other errors that indicate bugs or system-level conditions. The catch block's handling logic is designed for one type of error but is applied to all. Bugs are hidden behind generic error messages.

**Detection signals**:
- `except Exception`, `catch (Exception e)`, `catch (Throwable t)`
- `catch (...)` in C++
- Catch blocks that handle the caught exception generically (log and return a default) rather than with type-specific logic
- Broad catches wrapping large code blocks where only one line can throw the expected exception

**Severity**: MEDIUM for most code. HIGH in critical paths where masking a bug could cause data loss or security issues.

**Recommended alternative**: Catch the specific exceptions you expect. If you need a fallback for truly unexpected errors (e.g., in a top-level request handler), catch broadly at that single point and report the error rather than handling it.

**Example**:

Anti-pattern:
```python
def parse_config(path):
    try:
        with open(path) as f:
            data = json.load(f)
            return Config(
                host=data["host"],
                port=int(data["port"]),
                timeout=float(data["timeout"]),
            )
    except Exception:
        return Config.defaults()  # masks FileNotFoundError, KeyError,
                                  # ValueError, TypeError, etc.
```

Alternative:
```python
def parse_config(path):
    try:
        with open(path) as f:
            data = json.load(f)
    except FileNotFoundError:
        logger.info("Config file not found at %s, using defaults", path)
        return Config.defaults()
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in {path}: {e}") from e

    try:
        return Config(
            host=data["host"],
            port=int(data["port"]),
            timeout=float(data["timeout"]),
        )
    except (KeyError, ValueError, TypeError) as e:
        raise ConfigError(f"Invalid config values in {path}: {e}") from e
```

---

## Quick-Reference Summary

| Anti-Pattern | Category | Severity | Key Signal |
|---|---|---|---|
| God Object | Structural | HIGH | >500 lines, >20 methods, vague naming |
| Anemic Domain Model | Structural | MEDIUM | Data-only classes + separate service classes |
| Big Ball of Mud | Structural | HIGH | Circular deps everywhere, no layer boundaries |
| Premature Optimization | Behavioral | MEDIUM-HIGH | Complex code without benchmark justification |
| Stringly-Typed Code | Behavioral | MEDIUM-HIGH | String comparisons for finite value sets |
| Callback Hell | Behavioral | MEDIUM | >3 levels of nested callbacks |
| Golden Hammer | Behavioral | MEDIUM | Same pattern applied everywhere regardless of fit |
| Inappropriate Intimacy | Coupling | HIGH | Accessing private/internal fields across classes |
| Shotgun Surgery | Coupling | MEDIUM-HIGH | Single change touches 5+ unrelated files |
| Feature Envy | Coupling | MEDIUM | Method uses other object's data more than own |
| Circular Dependencies | Coupling | HIGH | Import cycles, deferred imports as workaround |
| Mutable Global State | State | HIGH | Global variables modified by multiple functions |
| Singleton Abuse | State | MEDIUM-HIGH | Singletons as convenient global access points |
| Hidden Side Effects | State | HIGH | Read-named functions that write state |
| Swallowed Exceptions | Error Handling | HIGH | Empty catch blocks, catch-and-ignore |
| Error Code/Exception Mixing | Error Handling | MEDIUM | Mixed return-value and exception error handling |
| Overly Broad Catch | Error Handling | MEDIUM-HIGH | `except Exception` / `catch (Throwable)` |
