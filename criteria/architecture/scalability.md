# Scalability

Reference checklist for the architecture reviewer's scalability analysis. This document complements the scalability-related criteria in `review-architecture.md` by providing detailed detection approaches, severity classification rationale, and remediation patterns for each category.

---

## 1. Algorithmic Complexity

### O(n^2)+ in Hot Paths

**Detection approach**:

- Nested loops iterating over collections that grow with input size or data volume
- `.filter()`, `.find()`, or `.includes()` called inside a loop over another collection
- String concatenation in a loop (O(n^2) in languages without optimized string builders)
- Repeated linear scans of arrays or lists where a set or map lookup would be O(1)
- Sorting within a loop (O(n * m log m))
- Regular expressions with catastrophic backtracking potential on user-controlled input

**Code patterns to flag**:

```
# Pattern: nested collection iteration
for item in items:                    # O(n)
    if item.id in other_list:         # O(m) per iteration -> O(n*m) total
        ...

# Pattern: repeated linear search
for order in orders:
    customer = find_customer(order.customer_id, customer_list)  # O(n) scan each time

# Pattern: string building
result = ""
for line in lines:
    result += line + "\n"             # O(n^2) in many languages
```

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| O(n^2)+ in a request-handling path where n is user-controlled or grows with data | HIGH |
| O(n^2)+ in a batch job or background process where n can grow beyond thousands | MEDIUM |
| O(n^2)+ in a startup or initialization path that runs once | LOW |
| O(n^2)+ where n is bounded to a small constant (< 100) by design | INFO |

**Remediation patterns**:

- Replace list membership checks with set or hash map lookups
- Pre-build index structures (dictionaries keyed by ID) before the loop
- Use string builders or array joins instead of concatenation
- Replace nested loops with hash joins or sort-merge approaches
- Consider algorithmic redesign: can the problem be solved in O(n log n) or O(n)?

### Nested Collection Iterations

**Detection approach**:

- Three or more levels of nested iteration (O(n^3))
- Cartesian product operations on collections that are not intentional cross-joins
- Graph traversal without visited tracking (potential exponential blowup)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| O(n^3) or worse in any path handling production data | HIGH |
| Unintentional cartesian product from nested iterations | MEDIUM |
| Graph traversal without cycle detection | HIGH (if input is not guaranteed acyclic) |

### Missing Index Structures

**Detection approach**:

- Repeated lookups by key in a list or array instead of a map/dictionary
- Sorting a collection to binary search it, when a tree or sorted structure maintained incrementally would be more efficient
- Filtering collections repeatedly by the same criteria without a pre-built index

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Repeated linear lookups in a hot path that could use a hash map | MEDIUM |
| Missing index on a growing dataset queried frequently | MEDIUM |
| One-time lookup in a small, bounded collection | INFO |

---

## 2. Unbounded Growth

### No-Limit Caches

**Detection approach**:

- In-memory dictionaries or maps used as caches with no eviction policy
- Cache implementations lacking TTL (time-to-live), max-size, or LRU eviction
- `@cache` or `@lru_cache` decorators with no `maxsize` parameter (Python-specific: `@cache` is unbounded)
- Application-level caches that grow proportionally with unique input values

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Unbounded cache in a long-running server process with user-controlled keys | HIGH |
| Unbounded cache in a short-lived process (CLI tool, batch script) | LOW |
| Cache with size limit but no TTL, risking stale data | MEDIUM |
| Cache with both size limit and TTL properly configured | INFO (positive) |

**Remediation patterns**:

- Add `maxsize` to LRU caches and choose a size based on expected cardinality
- Implement TTL-based expiration using a time-aware cache (e.g., `cachetools.TTLCache`, Guava `CacheBuilder`, Redis)
- Use an external cache (Redis, Memcached) for caches that must survive process restarts or be shared across instances
- Monitor cache hit rates and size to validate configuration

### Growing Queues

**Detection approach**:

- In-memory queues (lists, deques, channels) with no backpressure or capacity limit
- Producer-consumer patterns where the producer can outpace the consumer indefinitely
- Message queues without dead-letter configuration or max queue depth

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Unbounded in-memory queue in a request path where bursts can cause OOM | CRITICAL |
| Queue with no dead-letter handling, causing failed messages to retry forever | HIGH |
| Bounded queue but no monitoring or alerting on depth | MEDIUM |
| Queue with capacity limits, backpressure, and dead-letter handling | INFO (positive) |

**Remediation patterns**:

- Set explicit capacity limits on in-memory queues and define backpressure behavior (block, drop, reject)
- Configure dead-letter queues for messages that exceed retry limits
- Add monitoring on queue depth with alerts at capacity thresholds (70%, 90%)
- Consider external message brokers (RabbitMQ, Kafka, SQS) for durability and backpressure

### Accumulating State

**Detection approach**:

- Lists, maps, or sets that are appended to but never trimmed, rotated, or bounded
- Event listeners or callbacks registered without deregistration
- Temporary files created without cleanup
- Database connections opened without being returned to a pool or closed
- Session objects stored server-side without expiration

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Unbounded state accumulation in a production server process | HIGH |
| Resource leak (connections, file handles) under normal operation | HIGH |
| State accumulation bounded by session lifetime or request scope | MEDIUM |
| Accumulation in a test or development tool | LOW |

**Remediation patterns**:

- Use bounded data structures with automatic eviction
- Implement resource cleanup in `finally`/`defer`/`using`/`with` blocks
- Register shutdown hooks to clean up process-level resources
- Add periodic sweeps for orphaned temporary state

### Missing Pagination

**Detection approach**:

- API endpoints returning full result sets without limit/offset or cursor parameters
- Database queries without `LIMIT` clauses on tables that grow
- List endpoints that load all records into memory before serializing
- UI components fetching "all items" instead of a page at a time

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Public API endpoint returning unbounded results from a growing table | HIGH |
| Internal API without pagination on a table exceeding 10,000 rows | MEDIUM |
| Admin-only endpoint without pagination on a small, slow-growing table | LOW |

**Remediation patterns**:

- Implement cursor-based pagination for large or real-time datasets
- Implement offset/limit pagination for simpler use cases
- Set a server-side maximum page size even if the client does not specify one
- Return pagination metadata (total count, next cursor, has_more) in responses

---

## 3. Data Access

### N+1 Queries

**Detection approach**:

- Loop that executes a database query per iteration to fetch related records
- ORM lazy-loading traversals inside loops (e.g., accessing `order.customer.name` inside a `for order in orders` loop)
- GraphQL resolvers that fetch related entities one at a time without batching (missing DataLoader)
- REST API calls inside loops fetching related resources

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| N+1 in a user-facing request path where N grows with data | HIGH |
| N+1 in a background job or admin report | MEDIUM |
| N+1 where N is bounded to a small constant (< 10) and the query is fast | LOW |

**Remediation patterns**:

- Use eager loading / join fetching (`select_related`, `Include`, `JOIN`)
- Batch queries: collect IDs first, then fetch all related records in a single `WHERE IN` query
- Use DataLoader pattern for GraphQL (batches + caching within a request)
- Denormalize if the join is consistently needed and write frequency is low

### Over-Fetching

**Detection approach**:

- `SELECT *` in queries where only a few columns are needed
- ORM queries loading full object graphs including nested relationships not used by the caller
- API responses returning 30+ fields when the consumer uses 3-5
- Loading entire file contents into memory when only metadata is needed

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| SELECT * on a table with large TEXT/BLOB columns in a high-frequency path | HIGH |
| Full object graph loading with unused nested relationships | MEDIUM |
| Minor over-fetching of a few extra columns in a low-frequency path | LOW |

**Remediation patterns**:

- Select only required columns in queries
- Use projection DTOs or view models that request only needed fields
- Implement GraphQL or sparse fieldsets to let clients specify needed fields
- Use lazy loading for relationships that are only occasionally accessed

### Missing Indexes

**Detection approach**:

- Queries filtering on columns that are not part of a defined index (requires schema knowledge)
- `WHERE` clauses on foreign key columns without indexes
- `ORDER BY` on non-indexed columns in queries returning large result sets
- Full-text search using `LIKE '%term%'` instead of a full-text index
- Composite queries where the index column order does not match the query's filter order

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Missing index on a high-cardinality column queried in a hot path | HIGH |
| Missing index on a foreign key used in joins | MEDIUM |
| Missing index on a column queried only in batch or admin operations | LOW |

**Remediation patterns**:

- Add indexes on columns used in WHERE, JOIN, and ORDER BY clauses
- Use composite indexes for multi-column filters, with the most selective column first
- Add partial indexes for queries that consistently filter on a specific condition
- Use EXPLAIN/EXPLAIN ANALYZE to validate query plans

### Write Amplification

**Detection approach**:

- ORM saving entire objects when only one field changed (full-row updates)
- Document databases replacing the full document on every update
- Event sourcing projections that rebuild from scratch instead of incrementally updating
- Updating denormalized data in multiple locations without batching

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Full-document replacement on a large document updated frequently | HIGH |
| Full-row ORM updates where partial updates are supported and the row is wide | MEDIUM |
| Full-row updates on narrow tables with infrequent writes | LOW |

**Remediation patterns**:

- Use partial updates (`UPDATE SET column = value`, `$set` in MongoDB)
- Enable dirty tracking in the ORM to only update changed fields
- Use incremental projection updates instead of full rebuilds
- Batch related writes into single transactions

---

## 4. Async Patterns

### Blocking I/O in Request Paths

**Detection approach**:

- Synchronous HTTP calls, file reads, or database queries in async handler contexts
- `time.sleep()` or equivalent in request-handling code
- Synchronous file I/O in an event-loop-based framework (Node.js `fs.readFileSync`, Python `open()` in asyncio handler)
- DNS resolution blocking the event loop

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Blocking I/O in an event-loop framework's request handler | HIGH |
| Synchronous external API call in a web request path with no timeout | HIGH |
| Blocking I/O in a thread-per-request server (tolerable but suboptimal) | MEDIUM |
| Blocking I/O in a CLI tool or batch process | LOW |

**Remediation patterns**:

- Use async I/O libraries matching the framework (aiohttp, asyncpg, fs.promises)
- Offload blocking operations to a thread pool (`run_in_executor`, worker threads)
- Use non-blocking DNS resolution
- Add explicit timeouts to all external I/O calls

### Deferred Processing

**Detection approach**:

- Email sending, PDF generation, webhook delivery, or report generation executed synchronously in request handlers
- Operations that do not need to complete before responding to the user but are executed inline
- Long-running computations blocking HTTP responses

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Synchronous email/webhook delivery in a user-facing request adding seconds of latency | MEDIUM |
| Synchronous report generation blocking an API response for minutes | HIGH |
| Minor computation (< 100ms) that could be deferred but does not impact UX | INFO |

**Remediation patterns**:

- Move slow operations to a task queue (Celery, Sidekiq, Bull, SQS + workers)
- Use fire-and-forget patterns for non-critical notifications
- Return 202 Accepted with a job ID for long-running operations
- Implement webhook delivery with async retry queues

### Thread and Goroutine Management

**Detection approach**:

- `new Thread()` or `go func()` called without limit in a loop or per-request
- Thread pools with unbounded maximum sizes
- Missing error handling in spawned threads/goroutines (silent failures)
- No mechanism to wait for spawned work to complete on shutdown (goroutine leaks)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Unbounded thread/goroutine creation per request | HIGH |
| Thread pool with no maximum size and no monitoring | MEDIUM |
| Spawned goroutines with no error propagation | MEDIUM |
| Missing graceful shutdown for background workers | MEDIUM |

**Remediation patterns**:

- Use bounded thread/worker pools with configurable sizes
- Implement semaphore-based concurrency limiting
- Use structured concurrency patterns (task groups, errgroups)
- Add graceful shutdown with drain timeouts for in-flight work

---

## 5. Caching

### Missing Caches

**Detection approach**:

- Identical database queries or API calls executed multiple times within the same request
- Expensive computations (parsing, rendering, aggregation) repeated with the same inputs
- Configuration or reference data fetched from a database on every request
- External API calls with rate limits but no response caching

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Repeated expensive queries within a single request multiplying latency | HIGH |
| Reference data fetched from database on every request when it changes rarely | MEDIUM |
| Minor repeated computation with negligible cost | LOW |
| Well-implemented caching with appropriate TTL and invalidation | INFO (positive) |

**Remediation patterns**:

- Add request-scoped caching for data reused within a single request (identity map, context cache)
- Cache reference data with TTL matching its change frequency
- Implement read-through caching for external service responses
- Use memoization for pure function computations

### Invalidation Correctness

**Detection approach**:

- Cache entries that outlive the data they represent (no invalidation on write)
- Write operations that update the database but do not invalidate or update the cache
- Time-based expiration that is too long for the data's change frequency
- No cache invalidation strategy documented or implemented

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Cache serving stale data for mutable user-facing content (prices, inventory, permissions) | HIGH |
| Stale cache for non-critical display data (user profile images, analytics dashboards) | MEDIUM |
| Cache for immutable or append-only data (no invalidation needed) | INFO |

**Remediation patterns**:

- Invalidate on write: delete or update cache entries when the underlying data changes
- Use write-through caching to keep cache and database in sync
- Set TTL as a safety net even when using explicit invalidation
- Use cache versioning for complex invalidation scenarios

### Thundering Herd

**Detection approach**:

- Popular cache keys with high read rates and periodic expiration
- No locking or single-flight mechanism on cache miss
- All instances racing to rebuild the cache simultaneously on expiration
- Cache warm-up not implemented after deployments or restarts

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Thundering herd risk on a cache key backing a high-traffic endpoint | HIGH |
| Potential thundering herd on moderate-traffic endpoints | MEDIUM |
| Single-instance deployment where thundering herd is not possible | INFO |

**Remediation patterns**:

- Implement single-flight / lock-based cache rebuilding (only one request rebuilds, others wait)
- Use stale-while-revalidate: serve stale data while one request rebuilds in the background
- Stagger TTLs with jitter to avoid synchronized expiration
- Implement cache warming on deployment

### Key Design

**Detection approach**:

- Cache keys that do not include all parameters affecting the cached value (key collisions)
- Cache keys that include unnecessary parameters, reducing hit rates
- No namespace or prefix separation between different types of cached data
- User-specific data cached with keys that lack user identification

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Cache key collision causing users to see other users' data | CRITICAL |
| Missing parameter in cache key causing stale or incorrect results | HIGH |
| Overly specific cache keys reducing hit rates | MEDIUM |
| Well-designed cache key structure | INFO (positive) |

---

## 6. Resource Management

### Connection Pools

**Detection approach**:

- Database connections opened per request and closed after (`connect()` in request handler)
- HTTP clients created per request instead of reused
- Connection pool settings using framework defaults without explicit configuration
- No health checking or validation on pooled connections (stale connections)
- Pool exhaustion not handled (no timeout on pool acquisition, no monitoring)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| No connection pooling for database access in a web server | HIGH |
| Connection pool with no max size or no acquisition timeout | HIGH |
| Pool using defaults without tuning for expected load | MEDIUM |
| Pool properly configured with health checks and monitoring | INFO (positive) |

**Remediation patterns**:

- Use connection pool libraries appropriate to the database/protocol
- Configure min, max, and idle connection counts based on expected concurrency
- Set acquisition timeouts to fail fast rather than queue indefinitely
- Enable connection validation/health checks to detect stale connections
- Monitor pool utilization metrics

### File Handles

**Detection approach**:

- `open()` calls without corresponding `close()` or context manager (`with` statement)
- File handles stored in instance variables without cleanup on object destruction
- Exception paths that skip file handle closure
- Temporary file creation without cleanup in `finally` blocks

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| File handle leak in a request-handling path (accumulates per request) | HIGH |
| File handle leak in a one-shot operation | LOW |
| File handle properly managed with context managers / try-finally | INFO (positive) |

**Remediation patterns**:

- Use context managers (`with`, `using`, `try-with-resources`, `defer`)
- Implement `Closeable`/`AutoCloseable` interfaces and use try-with-resources
- Add finalizer logging to detect leaked handles in development
- Use temporary file utilities that handle cleanup automatically

### Timeouts

**Detection approach**:

- HTTP client calls without explicit timeout configuration
- Database queries with no statement timeout
- gRPC calls using default (infinite) deadline
- Socket operations without `SO_TIMEOUT` or equivalent
- External service calls where a timeout is set on the client but not on the server-side operation

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| No timeout on external service calls in a request path (cascade failure risk) | HIGH |
| No timeout on database queries (long query can hold connections indefinitely) | HIGH |
| Timeout set but unreasonably high (> 60s for a synchronous user-facing call) | MEDIUM |
| Timeouts properly configured with reasonable values | INFO (positive) |

**Remediation patterns**:

- Set explicit connect and read timeouts on all HTTP clients
- Configure statement timeouts at the database connection level
- Set gRPC deadlines on every call
- Implement circuit breakers for external services that combine timeout + failure threshold
- Document timeout values and their rationale

### Thread Pools

**Detection approach**:

- Thread pools with unbounded or excessively large maximum sizes
- Shared thread pools used for both CPU-bound and I/O-bound work
- No rejection policy configured (what happens when the pool is full?)
- Thread pools without monitoring (utilization, queue depth, rejection count)

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Unbounded thread pool creation in a production service | HIGH |
| Thread pool shared between fast and slow operations causing head-of-line blocking | MEDIUM |
| Thread pool with no rejection policy (defaults to caller-runs or abort) | MEDIUM |
| Thread pool with appropriate sizing, monitoring, and rejection handling | INFO (positive) |

**Remediation patterns**:

- Size thread pools based on workload type: CPU-bound (cores + 1), I/O-bound (cores * ratio of wait-to-compute time)
- Separate thread pools for different workload types to prevent head-of-line blocking
- Configure explicit rejection policies (reject, caller-runs, discard-oldest) based on business requirements
- Monitor thread pool metrics: active threads, queue depth, completed tasks, rejected tasks

---

## 7. Horizontal Scaling Blockers

### Local State

**Detection approach**:

- In-process caches (hash maps, LRU caches) storing data that must be consistent across instances
- Session data stored in memory (e.g., `HttpSession` in Java, in-process session stores)
- Rate limiting using local counters
- File uploads stored on local filesystem
- Scheduled jobs or cron tasks that would run on every instance

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Session state in memory in a service that needs horizontal scaling | HIGH |
| In-process cache for data requiring consistency across instances | HIGH |
| Local file storage for user uploads in a multi-instance deployment | HIGH |
| Local rate limiting that allows N * (instance count) requests | MEDIUM |
| In-process cache for immutable reference data (each instance caches independently, no consistency issue) | INFO |

**Remediation patterns**:

- Move session state to an external store (Redis, database)
- Replace in-process caches with distributed caches (Redis, Memcached)
- Use object storage (S3, GCS, Azure Blob) for file uploads
- Use distributed rate limiting (Redis-based token bucket, API gateway rate limiting)
- Use a job scheduler with leader election or external scheduling (cron service, Kubernetes CronJob)

### Sticky Sessions

**Detection approach**:

- Load balancer configuration using session affinity or sticky sessions
- Application code assuming requests from the same user hit the same server
- WebSocket connections without reconnection handling for server changes
- Server-side state accumulated across multiple requests from the same client

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Sticky sessions required due to in-memory state, preventing graceful scaling and rolling deployments | HIGH |
| Sticky sessions for WebSocket connections without failover capability | MEDIUM |
| Session affinity used as optimization but not required for correctness | LOW |

**Remediation patterns**:

- Externalize all state that creates the affinity requirement
- Implement stateless request handling with external session stores
- For WebSocket: implement reconnection with state recovery from external store
- Use consistent hashing at the load balancer if affinity is needed for caching efficiency (not correctness)

### Hardcoded Endpoints

**Detection approach**:

- Hardcoded hostnames, IP addresses, or port numbers in application code
- Service URLs constructed from string literals instead of configuration or service discovery
- Localhost assumptions (`127.0.0.1`, `localhost`) for services that will be external in production
- Hardcoded replica counts or instance identifiers

**Severity classification**:

| Condition | Severity |
|-----------|----------|
| Production service URLs hardcoded in application code | HIGH |
| Localhost references for services that are external in production | MEDIUM |
| Hardcoded ports that conflict in multi-instance deployment | MEDIUM |
| Development-only localhost references clearly separated by configuration profiles | LOW |

**Remediation patterns**:

- Externalize all service endpoints to configuration (environment variables, config files, service discovery)
- Use service discovery (DNS, Consul, Kubernetes service names) instead of static addresses
- Implement connection configuration that supports multiple backends
- Separate development and production configuration profiles
