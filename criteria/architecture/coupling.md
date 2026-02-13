# Coupling and Cohesion

Reference checklist for the architecture reviewer's coupling and cohesion analysis. This document complements the coupling-related criteria in `review-architecture.md` by providing detailed detection techniques, severity classification rationale, and recommended fix patterns for each category.

---

## 1. Dependency Direction

### Layer Violations

**How to detect**:

- Map the project's intended layer structure (presentation, business/domain, data/infrastructure) from directory layout, module naming, or documented architecture
- Check import statements: inner layers must not import from outer layers
- Look for domain model classes importing HTTP framework types, ORM session objects, or UI components
- Check for "shortcut" imports where a controller directly calls a repository, bypassing the service layer

**Common layer violation patterns**:

```
# Violation: domain model importing infrastructure
from app.infrastructure.email import SmtpClient    # domain should not know about SMTP

# Violation: data layer importing presentation
from app.api.serializers import UserResponse        # repository should not know about API shapes

# Violation: controller bypassing service layer
class OrderController:
    def create(self, request):
        self.order_repo.save(Order(...))            # skipping business validation in service
```

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Domain/business layer depending on infrastructure or presentation types | HIGH |
| Pervasive layer violations across multiple modules, making testing without full stack impossible | HIGH |
| Isolated shortcut (controller calling repository directly) with business logic still in service | MEDIUM |
| Utility module importing from a layer it should not know about | MEDIUM |
| Minor import crossing a boundary with no behavioral consequence | LOW |

**Recommended fix patterns**:

- Define interfaces/abstractions in the inner layer; implement them in the outer layer
- Use Dependency Inversion: the domain defines a `NotificationService` interface; infrastructure provides `SmtpNotificationService`
- Enforce layer boundaries with architectural tests (ArchUnit, dependency-cruiser, import-linter)
- Restructure directory layout to make layer boundaries explicit

### Circular Dependencies

**How to detect**:

- Trace import chains: if module A imports B and B imports A (directly or transitively), there is a cycle
- Use tooling: `madge` (JavaScript), `import-linter` (Python), `go vet` (Go prevents this at compile time), `dependency-cruiser` (JavaScript/TypeScript)
- Look for forward declarations, late imports, or runtime imports used to break cycles (these are symptoms, not fixes)
- Check for interface files created solely to break a cycle (another symptom)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Circular dependency causing initialization failures or deadlocks | CRITICAL |
| Circular dependency between major modules or packages | HIGH |
| Circular dependency between classes within the same module (less harmful, but still a design smell) | MEDIUM |
| Cycle broken by a well-placed interface extraction (acceptable if the interface is meaningful) | LOW |

**Recommended fix patterns**:

- Extract the shared concept into a third module that both depend on (dependency on shared abstraction)
- Apply Dependency Inversion: one module defines an interface, the other implements it
- Merge tightly coupled modules if they represent a single cohesive concept
- Use event-based decoupling for notification-style dependencies
- Restructure so dependencies flow in one direction (typically toward the domain core)

### Upward Dependencies

**How to detect**:

- Library or utility code importing application-specific types
- Shared/common modules referencing specific feature modules
- Infrastructure packages depending on business logic packages
- Generic frameworks or toolkits importing project-specific configuration

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Shared library depending on application types, preventing reuse | HIGH |
| Utility module importing from a specific feature, coupling unrelated features | MEDIUM |
| Minor upward reference in a utility helper that could be easily inverted | LOW |

**Recommended fix patterns**:

- Pass application-specific behavior to the utility via callbacks, interfaces, or generics
- Move the application-specific logic out of the utility into the calling code
- Use the plugin or extension pattern: the utility defines extension points, the application provides implementations

---

## 2. Module Boundaries

### Mixed Concerns

**How to detect**:

- A single module/package that contains files addressing unrelated business domains (e.g., `utils/` containing auth helpers, string formatters, database connectors, and email templates)
- Classes with methods spanning multiple responsibility areas (validation + persistence + notification in one class)
- Package names like `common`, `helpers`, `utils`, `misc` that grow without bound
- Files that import from many unrelated packages (high import diversity signals mixed concerns)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| "God" module with 20+ files covering unrelated domains | HIGH |
| Class with 10+ injected dependencies spanning multiple concerns | HIGH |
| `utils` package with 5-10 unrelated utility groups that should be separate modules | MEDIUM |
| Minor concern mixing in a single file (two related but distinct responsibilities) | LOW |

**Recommended fix patterns**:

- Decompose god modules into focused packages organized by business domain or technical concern
- Apply the Single Responsibility Principle: each module should have one reason to change
- Create explicit sub-packages within large modules to group related functionality
- Extract cross-cutting utilities into purpose-named packages (`retry`, `validation`, `serialization`) rather than a monolithic `utils`

### Over-Exported APIs

**How to detect**:

- Modules that export (make public) most or all of their internal types and functions
- `__init__.py` files that re-export everything from submodules
- Packages without clear public vs. internal separation (no `internal/` directory, no underscore-prefixed modules, no explicit `exports` in `package.json`)
- API surface area that includes implementation helpers alongside core abstractions

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Internal implementation types used across module boundaries, creating coupling to unstable internals | HIGH |
| No distinction between public API and internal implementation in a shared library | MEDIUM |
| A few extra exported symbols that are not intended for external use | LOW |
| Well-defined public API with internal details properly hidden | INFO (positive) |

**Recommended fix patterns**:

- Use language-specific visibility controls (package-private in Java, `internal/` in Go, `_` prefix in Python, `exports` field in `package.json`)
- Create an explicit public API surface: an `index` or `__init__` file that exports only the intended interface
- Document which types and functions are part of the stable public API
- Use barrel files or facade modules to present a curated API

### Internal Detail Leakage

**How to detect**:

- Public method signatures that include internal types (returning a database cursor, ORM model, or framework-specific request object through the module boundary)
- Configuration details (database table names, queue names, internal service URLs) exposed in a module's public interface
- Error types that expose internal stack traces or implementation-specific error codes to external consumers
- Method parameters that accept internal data structures instead of stable abstractions

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Database model types exposed in a public API consumed by multiple teams | HIGH |
| Internal error details (stack traces, SQL) exposed to end users | HIGH (also a security concern) |
| Internal types leaked through method signatures but only consumed by adjacent modules | MEDIUM |
| Minor internal type in a method signature that could be replaced with a standard type | LOW |

**Recommended fix patterns**:

- Define DTOs or value objects at the module boundary that translate to/from internal types
- Map internal errors to domain-specific error types before crossing module boundaries
- Use anti-corruption layers between bounded contexts
- Apply the Interface Segregation Principle: expose only what consumers need

---

## 3. Dependency Injection

### Hardcoded Instantiation

**How to detect**:

- Classes that create their own dependencies using `new` or direct constructor calls for services (not for value objects or data structures)
- Static method calls to obtain service instances (`DatabaseService.getInstance()`)
- Import-time module-level instantiation of services that should be configurable
- Test files that cannot substitute dependencies because they are hardcoded

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Hardcoded instantiation of external service clients (database, HTTP, message queue) making testing impossible without the actual service | HIGH |
| Hardcoded instantiation of business logic services preventing unit testing of dependent classes | MEDIUM |
| Hardcoded instantiation of simple internal utilities with no state or external dependencies | LOW |
| Value objects and data structures instantiated directly (this is correct, not a finding) | N/A |

**Recommended fix patterns**:

- Accept dependencies through constructor parameters (constructor injection)
- Use a DI container or framework if the language/ecosystem supports it (Spring, .NET DI, Python dependency-injector, Go wire)
- For languages without DI frameworks, use factory functions that accept dependencies as parameters
- Separate object creation from object use: create in the composition root, use everywhere else

### Service Locator Anti-Pattern

**How to detect**:

- Global registry or container queried at runtime to obtain dependencies (`Container.resolve(UserService)`)
- Static accessor methods that look up services from a global map
- Ambient context pattern where dependencies are obtained from thread-local or request-scoped globals
- Dependencies not visible in constructor or method signatures (hidden dependencies)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Pervasive service locator usage making dependency graphs invisible and testing difficult | HIGH |
| Service locator used in a few places alongside proper injection | MEDIUM |
| Service locator used only in the composition root to resolve top-level dependencies (acceptable) | INFO |

**Recommended fix patterns**:

- Convert service locator calls to constructor injection
- Make dependencies explicit in constructor signatures
- If a DI container is used, use it only in the composition root; inject concrete instances everywhere else
- Replace ambient context with explicit parameter passing

### Constructor Bloat

**How to detect**:

- Constructors or initialization methods accepting more than 5-7 parameters
- Classes requiring many injected services to function
- Constructor parameters that include a mix of configuration values and service dependencies
- Parameter objects created solely to reduce constructor parameter count (may mask the real problem)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Constructor with 10+ dependencies, indicating the class does too much | HIGH |
| Constructor with 6-9 dependencies spanning multiple concerns | MEDIUM |
| Constructor with 5-7 focused dependencies in a logically complex domain | LOW |
| Constructor with 3-4 dependencies that are cohesively related | INFO (positive) |

**Recommended fix patterns**:

- Decompose the class: split it into smaller classes with fewer responsibilities
- Extract groups of related dependencies into a collaborator object (Facade pattern)
- Separate configuration parameters from service dependencies
- Apply the Interface Segregation Principle: does this class need all methods of each dependency, or only a subset?

---

## 4. Fan-Out and Fan-In

### Over-Connected Modules

**How to detect**:

- Count import statements per file or module: more than 10-15 direct dependencies signals high fan-out
- Modules that import from across many different packages or domains
- A change in the over-connected module ripples to many consumers (high fan-in combined with high fan-out)
- Visualization tools (dependency graphs) showing one module at the center with spokes to many others

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Module with 15+ direct dependencies spanning multiple domains | HIGH |
| Module with 10-15 dependencies from related but distinct areas | MEDIUM |
| Module with high import count but all imports are from the same cohesive package | LOW |

**Recommended fix patterns**:

- Decompose the module into smaller, focused modules
- Introduce mediator or orchestrator patterns to reduce direct dependencies
- Apply facade pattern: depend on a facade that encapsulates a subsystem rather than individual classes
- Re-examine module boundaries: the module may be at the wrong level of abstraction

### God Classes

**How to detect**:

- Classes exceeding 500-1000 lines with methods spanning multiple responsibilities
- Classes with 20+ public methods covering different functional areas
- Classes that are modified by every feature branch (high change frequency from diverse causes)
- Class names with vague suffixes: `Manager`, `Handler`, `Processor`, `Helper`, `Service` that do not narrow the scope

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| God class in a critical business path that blocks independent team development | HIGH |
| God class in a utility or infrastructure module with stable behavior | MEDIUM |
| Class that is large but cohesive (all methods operate on the same core data structure) | LOW |
| Well-decomposed classes with clear, narrow responsibilities | INFO (positive) |

**Recommended fix patterns**:

- Extract method groups into collaborator classes
- Apply the Extract Class refactoring: identify clusters of fields and methods that operate together
- Use the Strangler Fig approach for large classes: create new focused classes and delegate to them, migrating callers incrementally
- Consider domain-driven decomposition: align classes with bounded contexts

### Shotgun Surgery

**How to detect**:

- A single logical change (adding a field, changing a business rule) requires modifying five or more files across different modules
- Feature additions that touch controller, service, repository, model, DTO, mapper, validator, and test files in different packages
- Configuration changes that require updates in multiple unrelated locations
- Duplicated switch statements or conditional logic across multiple classes that must be updated in lockstep

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Routine changes requiring 10+ file modifications across unrelated modules | HIGH |
| Changes requiring 5-10 modifications in related but dispersed locations | MEDIUM |
| Changes requiring modifications in 3-4 files within the same module (may be normal for a layered architecture) | LOW |

**Recommended fix patterns**:

- Consolidate related behavior into a single module or class (move method, move field)
- Use polymorphism to replace duplicated switch statements (Strategy pattern)
- Extract the cross-cutting change into a shared location (configuration, metadata, schema definition)
- Apply the information expert principle: place behavior where the data is

---

## 5. Shared State

### Global Mutable State

**How to detect**:

- Module-level variables that are mutated after initialization
- Static mutable fields accessed from multiple classes
- Global configuration objects modified at runtime
- `global` keyword usage (Python) or global variable assignments
- Mutable class variables shared across all instances (Python: mutable default arguments, class-level lists/dicts)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Global mutable state accessed from multiple threads without synchronization | CRITICAL |
| Global mutable state creating hidden coupling between modules | HIGH |
| Module-level mutable state used within a single module for caching, with documented behavior | MEDIUM |
| Global constants (read-only after initialization) | INFO (not a finding) |

**Recommended fix patterns**:

- Convert global mutable state to explicitly passed parameters or injected dependencies
- Use immutable configuration objects loaded once at startup
- If global state is genuinely needed, protect it with synchronization and document the concurrency contract
- Use thread-local storage for state that must be global within a thread but isolated between threads
- Apply the Singleton pattern with proper thread safety if a single mutable instance is required

### Thread-Unsafe Sharing

**How to detect**:

- Data structures shared between threads without locks, atomics, or concurrent collection types
- Check-then-act patterns without synchronization (TOCTOU: time-of-check to time-of-use)
- Lazy initialization of shared state without double-checked locking or equivalent
- Collections modified while being iterated by another thread
- Mutable objects passed between threads via shared references without defensive copying

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Race condition on shared mutable state in a production code path | CRITICAL |
| Potential race condition on shared state that could cause data corruption | HIGH |
| Unsynchronized access to shared state in a low-concurrency context (unlikely but possible race) | MEDIUM |
| Thread-safe data structures and synchronization properly used | INFO (positive) |

**Recommended fix patterns**:

- Use concurrent data structures (`ConcurrentHashMap`, `sync.Map`, thread-safe queues)
- Apply locks at the appropriate granularity (not too coarse, not too fine)
- Use immutable objects and value types to eliminate shared mutable state
- Apply the actor model or message-passing patterns to avoid shared state entirely
- Use atomic operations for simple counters and flags

### Implicit Ordering Requirements

**How to detect**:

- Functions or methods that must be called in a specific order but the order is not enforced by the API
- Initialization sequences where skipping a step causes silent incorrect behavior (not an error)
- State machines without explicit state tracking (behavior depends on which methods were called previously)
- Comments like "must call init() before use" or "call after configure()"

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Implicit ordering causing silent data corruption or incorrect results | HIGH |
| Implicit ordering causing exceptions or failures (at least the problem is visible) | MEDIUM |
| Ordering requirement documented and enforced by runtime checks | LOW |
| Builder pattern or fluent API enforcing order through types (compile-time safety) | INFO (positive) |

**Recommended fix patterns**:

- Use the Builder pattern to enforce construction order through the type system
- Implement state machine with explicit state tracking and transitions
- Add runtime guards that throw if preconditions are not met (`if (!initialized) throw`)
- Redesign the API so that all required state is provided at construction time
- Use the typestate pattern (in languages that support it) to encode valid operation sequences in the type system

---

## 6. API Design

### Leaky Abstractions

**How to detect**:

- Interface methods that include implementation-specific parameters (database connection passed through a business interface, HTTP headers in a domain method)
- Abstractions that require callers to understand the underlying implementation to use correctly
- Error types from lower layers propagated through interfaces (SQL exceptions thrown by a repository interface)
- Method names that reference implementation technology (`findBySQL`, `sendViaKafka`)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Leaky abstraction forcing all consumers to understand implementation details | HIGH |
| Implementation-specific error types crossing abstraction boundaries | MEDIUM |
| Minor naming references to implementation that do not affect behavior | LOW |
| Clean abstractions that hide implementation details effectively | INFO (positive) |

**Recommended fix patterns**:

- Define interface methods in terms of the domain, not the implementation
- Translate implementation-specific exceptions to domain-specific exceptions at the boundary
- Use adapters to convert between implementation types and domain types
- Review interfaces by asking: "Could I swap the implementation without changing any caller?"

### Breaking Changes

**How to detect**:

- Method signature changes in public interfaces (added required parameters, changed return types, removed methods)
- Enum or constant value changes that affect serialization or storage
- Behavioral changes that alter postconditions or side effects without updating the contract
- Field renames or type changes in serialized data structures (JSON, Protocol Buffers, database schemas)
- Removed or renamed API endpoints

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Breaking change to a public API consumed by external clients without version bump | CRITICAL |
| Breaking change to an internal API consumed by multiple services | HIGH |
| Breaking change to an internal API consumed by one known consumer | MEDIUM |
| Additive change to an API (new optional fields, new endpoints) | INFO |

**Recommended fix patterns**:

- Version APIs (URL path versioning, header versioning, or content negotiation)
- Use additive-only changes: add new fields, endpoints, or methods without removing existing ones
- Deprecate before removing: mark as deprecated in one release, remove in a later release
- For serialized data: use backward-compatible evolution (optional fields, default values)
- For database schemas: use expand-contract migrations (add new column, migrate data, remove old column)

### Missing Versioning

**How to detect**:

- Public REST APIs without version identifiers in URL or headers
- gRPC services without package versioning
- Shared libraries published without semantic versioning
- Database schemas without migration versioning
- No changelog or breaking change documentation

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| External-facing API with no versioning strategy and active consumers | HIGH |
| Internal API with no versioning and multiple consumer teams | MEDIUM |
| Internal API with a single known consumer and informal coordination | LOW |
| Well-versioned API with semantic versioning and changelog | INFO (positive) |

**Recommended fix patterns**:

- Adopt a versioning scheme before the first external consumer (retrofitting is expensive)
- Use semantic versioning for libraries (MAJOR.MINOR.PATCH)
- Use URL path or header versioning for REST APIs
- Maintain a changelog documenting breaking changes per version
- Automate compatibility checks in CI (contract tests, schema compatibility tools)

### Error Contract Clarity

**How to detect**:

- Functions or API endpoints that can fail but do not document or type their error cases
- Catch-all error handlers that return generic error messages losing context
- Inconsistent error formats across endpoints (some return `{error: "msg"}`, others return `{message: "msg", code: 123}`)
- Error codes that are ad-hoc strings rather than a documented enumeration
- APIs that return 500 for client errors or 200 for errors

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| No error contract on a public API, forcing consumers to guess failure modes | HIGH |
| Inconsistent error formats across endpoints in the same API | MEDIUM |
| Generic error messages that lose debugging context | MEDIUM |
| Errors documented with stable codes but minor formatting inconsistencies | LOW |
| Well-documented error contract with stable error codes and examples | INFO (positive) |

**Recommended fix patterns**:

- Define a standard error response format for the entire API (RFC 7807 Problem Details is a good starting point)
- Enumerate error codes and document them as part of the API contract
- Use typed error/result types in the codebase (Result/Either types, typed exceptions, error enums)
- Map internal errors to stable external error codes at the API boundary
- Include correlation IDs in error responses for traceability
- Return appropriate HTTP status codes (4xx for client errors, 5xx for server errors)
