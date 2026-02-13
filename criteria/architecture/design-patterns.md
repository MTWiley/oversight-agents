# Design Patterns

Reference checklist for the architecture reviewer's design pattern analysis. This document complements the pattern-related criteria in `review-architecture.md` by providing detailed detection guidance, severity classification rationale, and remediation patterns for each category.

---

## 1. Creational Patterns

### Factory and Abstract Factory

**When to recommend**:

- Object creation logic is duplicated across multiple call sites, each constructing the same type with similar but context-dependent parameters
- Client code needs to create objects from a family of related types without coupling to concrete classes
- Construction requires multi-step initialization that should not be repeated (database connections, HTTP clients with retry/timeout configuration)

**When it is overkill**:

- Only one concrete type exists and no additional types are foreseeable
- The constructor is trivial (two or three parameters, no conditional logic)
- The factory would simply delegate to `new` with no additional logic

**Signs of misuse**:

- `SimpleFactory` that wraps a single constructor call with no branching or configuration
- Factory methods that always return the same concrete type and never vary
- Factory hierarchies mirroring class hierarchies one-to-one without adding decision logic

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Scattered complex construction logic with divergent configurations across call sites | MEDIUM |
| Factory wrapping a trivial constructor, adding unnecessary indirection | LOW |
| Missing factory causing duplicated multi-step initialization in production paths | MEDIUM |
| Factory recommended but purely a readability improvement | INFO |

### Builder

**When to recommend**:

- Object construction involves many optional parameters (typically more than five), making constructor signatures unwieldy
- The same construction process must create different representations
- Immutable objects require step-by-step assembly

**When it is overkill**:

- The object has fewer than four parameters, all required
- A language provides named parameters or data classes that already solve the readability problem (e.g., Kotlin data classes, Python dataclasses with defaults)

**Signs of misuse**:

- Builder that sets every field in sequence with no optional or conditional steps
- Builder used where a simple constructor or factory method would suffice
- Builder with a single `build()` path and no variation

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Missing builder causing error-prone multi-parameter construction in critical paths | MEDIUM |
| Unnecessary builder adding boilerplate with no readability gain | LOW |

### Singleton

**When to recommend**:

- A single shared resource genuinely must exist once per process (connection pool, hardware interface, process-wide configuration loaded once at startup)
- The singleton is stateless or read-only after initialization

**When it is overkill**:

- Almost always. Singleton is the most overused pattern and should be the last resort, not the default.

**Signs of misuse**:

- Mutable global singleton used to share state between unrelated components (hidden coupling)
- Singleton masking a dependency injection need (components reach for `Instance.get()` instead of receiving the dependency explicitly)
- Singleton preventing testability because tests cannot substitute or reset the instance
- Multiple singletons depending on initialization order

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Mutable singleton creating hidden coupling across module boundaries | HIGH |
| Singleton blocking testability in critical business logic | MEDIUM |
| Singleton used for a genuinely single-instance resource, documented and thread-safe | INFO (positive) |
| Singleton where dependency injection would be cleaner, but harm is limited | LOW |

---

## 2. Structural Patterns

### Adapter

**When to recommend**:

- Third-party API types are used directly throughout the codebase, creating vendor lock-in
- An interface change in an external dependency would require modifications across many files
- Two subsystems need to communicate but have incompatible interfaces

**When it is overkill**:

- The external dependency is stable, well-abstracted, and used in only one module
- The project is a short-lived script or prototype where adaptability is not a concern

**Signs of misuse**:

- Adapter that simply re-exports every method of the adaptee with identical signatures (no-op adapter)
- Adapter layers stacked on top of each other, creating telescoping indirection

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Third-party types leaked across module boundaries with no isolation | MEDIUM |
| No-op adapter adding indirection without value | LOW |
| Well-placed adapter isolating a volatile external dependency | INFO (positive) |

### Facade

**When to recommend**:

- Client code must call multiple subsystem objects in a specific sequence to accomplish a single logical operation
- A complex subsystem needs a simplified interface for common use cases
- Module boundary clarity requires a single entry point

**When it is overkill**:

- The subsystem has only one or two components with simple interfaces
- The facade would expose the same interface as the single class it wraps

**Signs of misuse**:

- "Facade" that grows to expose every method of every subsystem class, becoming a god class
- Multiple facades over the same subsystem with overlapping responsibilities

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Complex subsystem with no simplified entry point, causing client code to embed orchestration logic | MEDIUM |
| Facade that has become a god class with 30+ methods | MEDIUM |
| Unnecessary facade wrapping a simple component | LOW |

### Decorator

**When to recommend**:

- Cross-cutting behavior (logging, caching, retry, metrics, authorization) needs to be added to existing functionality without modifying it
- Multiple optional behaviors can be composed in different combinations

**When it is overkill**:

- Only one combination of behaviors is ever used (just put them in the class directly)
- The decorator stack becomes deep enough that debugging through the layers is harder than understanding a single class

**Signs of misuse**:

- Decorator chain exceeding three or four layers, creating stack-trace noise and debugging difficulty
- Decorator that modifies core behavior rather than wrapping it, violating the single responsibility of decoration

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Cross-cutting concern (retry, auth, logging) copy-pasted across classes instead of decorated | MEDIUM |
| Deep decorator chain obscuring behavior | LOW |
| Well-composed decorator for a cross-cutting concern | INFO (positive) |

### Proxy

**When to recommend**:

- Access control, lazy initialization, or remote call abstraction is needed without changing client code
- Expensive resources should be loaded only on demand

**When it is overkill**:

- The proxied object is cheap to create and access control is handled elsewhere

**Signs of misuse**:

- Proxy adding latency or complexity to every access when the optimization applies to a small minority of cases
- "Smart" proxy accumulating too many responsibilities (caching + logging + auth + validation in one proxy)

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Missing proxy allowing unrestricted access to a sensitive resource | HIGH (context-dependent) |
| Proxy overhead in a hot path with no measurable benefit | LOW |

---

## 3. Behavioral Patterns

### Strategy

**When to recommend**:

- Long `if/else` or `switch` chains dispatch on type, mode, or configuration to select different behavior
- New variations of a behavior require modifying existing code rather than adding new classes
- Algorithm selection needs to change at runtime

**Detection of missed opportunities**:

- Functions with a `type` or `mode` parameter controlling which branch executes
- Multiple methods in a class that differ only in one algorithmic step
- Comments like "add new case here" near switch statements

**When it is overkill**:

- Only two branches exist and a third is unlikely
- The branching is a simple boolean toggle, not a family of algorithms

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Switch chain in a request-handling path that grows with each new type, violating open/closed principle | MEDIUM |
| Two-branch conditional that could be a strategy but gains nothing from it | INFO |
| Strategy pattern applied to a single-algorithm case, adding unnecessary indirection | LOW |

### Observer / Event

**When to recommend**:

- One component directly calls notification methods on multiple other components (tight notification coupling)
- Adding a new subscriber requires modifying the publisher
- Multiple independent reactions to the same state change are implemented in a single method

**When it is overkill**:

- Only one subscriber exists and the relationship is stable
- Event ordering matters and the observer pattern would make ordering implicit and harder to reason about
- The system is small enough that direct method calls are clearer

**Signs of misuse**:

- Event chains that are difficult to trace (event A triggers B triggers C), creating debugging nightmares
- Observers modifying the subject's state, creating feedback loops
- Event systems used for control flow within a single module where direct calls are simpler

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tight publisher-subscriber coupling requiring publisher changes for each new subscriber | MEDIUM |
| Untraceable event chains causing operational incidents | HIGH |
| Observer pattern applied to a stable one-to-one relationship | LOW |

### Command

**When to recommend**:

- Operations need to be queued, logged, undone, or replayed
- An action must be parameterized and passed around as a first-class concept
- Macro recording or audit trail functionality is needed

**When it is overkill**:

- Simple direct method invocations with no undo, queue, or logging requirement
- The language supports first-class functions, making the pattern's class ceremony unnecessary

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Operations executed directly with no auditability where regulatory requirements demand it | HIGH |
| Missing command abstraction causing duplicated undo logic | MEDIUM |
| Command pattern used where a simple function reference suffices | LOW |

### Template Method

**When to recommend**:

- Multiple classes share the same algorithm skeleton but differ in specific steps
- Copy-pasted methods that differ only in one or two sections

**When it is overkill**:

- Only one subclass exists
- The varying steps are better expressed as strategy injection (composition over inheritance)

**Signs of misuse**:

- Template method with so many hook points that subclasses must override nearly every step
- Deep inheritance hierarchies built around template methods

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Duplicated algorithm skeletons across multiple classes with diverging bug fixes | MEDIUM |
| Template method with excessive hook points, creating a fragile base class | MEDIUM |
| Template method suggested where composition would be cleaner | INFO |

---

## 4. Architectural Patterns

### MVC / MVP / MVVM

**Consistency checks**:

- Identify which pattern the project uses and verify consistent application across all features
- Flag controllers/presenters/view-models that contain business logic belonging in the model/service layer
- Flag views that directly access data sources, bypassing the controller/presenter layer
- Check that models do not depend on view or controller types

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Business logic in controllers/views making it untestable without UI | MEDIUM |
| Inconsistent application: some features use MVC, others bypass it | MEDIUM |
| Minor layer bleed in a single file | LOW |

### Repository / Data Access Layer

**Consistency checks**:

- All data access should go through repository interfaces, not ad-hoc queries scattered in business logic
- Repository methods should return domain objects, not raw database types
- Check for queries written directly in service classes or controllers

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| SQL/ORM queries in controller or presentation code | MEDIUM |
| Inconsistent data access: some entities use repositories, others use direct queries | MEDIUM |
| Repository returning database-specific types through its interface | LOW |

### CQRS (Command Query Responsibility Segregation)

**Consistency checks**:

- If CQRS is adopted, verify read and write models are actually separated
- Check for query operations that mutate state (violates CQS principle)
- Verify that commands do not return data beyond acknowledgment

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Partial CQRS adoption with some paths bypassing the separation, creating data inconsistency risk | HIGH |
| Query methods with side effects | MEDIUM |
| CQRS applied where simple CRUD would suffice | LOW |

### Event Sourcing

**Consistency checks**:

- All state changes should be recorded as events, not direct state mutations
- Event schema evolution and versioning must be planned
- Projections (read models) must be rebuildable from the event log
- Check for mutations that bypass the event log

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| State mutations bypassing the event log, causing divergence between event history and actual state | CRITICAL |
| Missing event versioning strategy | HIGH |
| Event sourcing applied to a simple CRUD domain with no audit or replay requirement | MEDIUM |

---

## 5. Anti-Patterns

### Unnecessary Abstraction Layers

**Detection**:

- Interface with exactly one implementation and no test double, created "for future flexibility"
- Wrapper classes that delegate every method to the wrapped class with no added behavior
- Three or more layers of indirection between a caller and the actual logic

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Excessive abstraction increasing cognitive load across the codebase | MEDIUM |
| Single unnecessary wrapper in a non-critical path | LOW |
| Premature abstraction obscuring what the code actually does | MEDIUM |

### Pattern for Pattern's Sake

**Detection**:

- Design pattern applied where a simpler construct (function, conditional, direct call) would be equally clear and more concise
- Pattern names in class names without the pattern solving an identifiable problem (e.g., `UserStrategyFactoryBuilder`)
- Pattern application that increases the number of files/classes without reducing complexity

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Gratuitous pattern use significantly increasing codebase size and onboarding difficulty | MEDIUM |
| Minor over-engineering in a single component | LOW |
| Noting a pattern that is correctly applied | INFO (positive) |

### Design Pattern Cargo Cult

**Detection**:

- Patterns copied from tutorials without adaptation to the project's actual needs
- Gang of Four patterns applied in a language where idiomatic alternatives exist (e.g., full Strategy class hierarchy in Python where a dictionary of functions suffices)
- Pattern terminology used in naming but the implementation does not actually follow the pattern's structure or intent

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Cargo cult pattern masking actual design problems | MEDIUM |
| Incorrect pattern implementation causing confusion | MEDIUM |
| Minor naming mismatch between pattern intent and implementation | LOW |

---

## 6. Separation of Concerns

### Layer Violations

**Detection**:

- Presentation layer importing persistence layer types (skipping business logic)
- Business logic directly formatting HTTP responses or HTML
- Data access code containing validation or business rules
- Infrastructure code (logging framework, message broker client) imported in domain model classes

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Pervasive layer violations making business logic untestable | HIGH |
| Isolated layer violation in a single file | MEDIUM |
| Minor import that crosses a boundary but has limited blast radius | LOW |

### Cross-Cutting Concern Consistency

**Detection**:

- Logging implemented ad-hoc in some modules and via middleware/decorators in others
- Authentication checked manually in some handlers and via middleware in others
- Metrics collection inconsistent: some endpoints instrumented, others not
- Error handling approach varies across modules (some throw, some return error codes, some use Result types)

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Inconsistent auth checking, causing some paths to lack protection | HIGH |
| Inconsistent logging making incident investigation unreliable | MEDIUM |
| Inconsistent error handling style across modules | MEDIUM |
| Minor inconsistency in metrics collection | LOW |
| Consistent cross-cutting concern implementation worth preserving | INFO (positive) |
