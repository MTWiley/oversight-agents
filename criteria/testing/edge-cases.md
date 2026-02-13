# Edge Cases Reference

This reference provides detailed criteria for evaluating whether test suites cover critical edge cases -- boundary values, null handling, unusual inputs, concurrency, error paths, and security-adjacent scenarios. Missing edge case coverage is the most common cause of production incidents that "could have been caught by a test."

The `testing-reviewer` agent uses this file as a lookup when evaluating edge case coverage. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Boundary Value Analysis

Boundary value analysis (BVA) tests the edges and limits of input domains, where off-by-one errors, overflow, and unexpected behavior are most likely to occur.

### Empty / Zero / Negative / Single-Element

**Rule**: Every function that accepts a collection, number, or string should be tested with empty, zero, negative (where applicable), and single-element inputs.

**Detection pattern -- missing boundary tests**:

```python
# Function under test
def calculate_average(numbers):
    return sum(numbers) / len(numbers)

# Existing tests (common case only)
def test_average_of_several_numbers():
    assert calculate_average([10, 20, 30]) == 20.0

# Missing edge cases:
def test_average_of_empty_list():
    with pytest.raises(ZeroDivisionError):
        calculate_average([])

def test_average_of_single_element():
    assert calculate_average([42]) == 42.0

def test_average_of_negative_numbers():
    assert calculate_average([-10, -20, -30]) == -20.0

def test_average_of_mixed_positive_and_negative():
    assert calculate_average([-10, 10]) == 0.0
```

```javascript
// Function under test
function findMax(arr) {
  return Math.max(...arr);
}

// Missing edge cases
test('findMax with empty array returns -Infinity', () => {
  expect(findMax([])).toBe(-Infinity);  // JavaScript quirk
});

test('findMax with single element returns that element', () => {
  expect(findMax([42])).toBe(42);
});

test('findMax with all identical elements', () => {
  expect(findMax([5, 5, 5])).toBe(5);
});

test('findMax with negative numbers', () => {
  expect(findMax([-3, -1, -7])).toBe(-1);
});
```

```go
func TestFindMax(t *testing.T) {
    tests := []struct {
        name    string
        input   []int
        want    int
        wantErr bool
    }{
        {"empty slice", []int{}, 0, true},
        {"single element", []int{42}, 42, false},
        {"all negative", []int{-3, -1, -7}, -1, false},
        {"all identical", []int{5, 5, 5}, 5, false},
        {"mixed positive and negative", []int{-10, 0, 10}, 10, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := FindMax(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}
```

### Numeric Boundaries

**Rule**: Numeric inputs should be tested at the exact boundary, one below, and one above.

| Boundary | Test Values | Why |
|----------|-------------|-----|
| Zero | -1, 0, 1 | Division by zero, sign change, index underflow |
| Array length | len-1, len, len+1 | Off-by-one, out-of-bounds |
| MAX_INT / MIN_INT | MAX-1, MAX, MAX+1 | Overflow, wraparound |
| Configured limits | limit-1, limit, limit+1 | Off-by-one in limit enforcement |
| Float precision | 0.1 + 0.2 vs 0.3 | Floating-point representation errors |

```java
// Integer overflow testing
@Test
void addLargeNumbers_handlesOverflow() {
    // int max is 2,147,483,647
    assertThrows(ArithmeticException.class, () -> {
        Math.addExact(Integer.MAX_VALUE, 1);
    });
}

@Test
void calculateTotal_handlesLargeItemCount() {
    // 100,000 items at $99.99 each -- does the total overflow?
    List<Item> items = Collections.nCopies(100_000, new Item("widget", 9999L));
    long total = calculateTotal(items);
    assertEquals(999_900_000L, total);  // Use long, not int
}
```

```python
# Float precision edge case
def test_float_comparison():
    # This is a common bug: 0.1 + 0.2 != 0.3 in floating point
    result = calculate_sum([0.1, 0.2])
    assert result == pytest.approx(0.3)  # Use approx, not ==

def test_large_number_precision():
    # Large numbers can lose precision in float
    result = process_amount(9_999_999_999_999.99)
    assert isinstance(result, Decimal)  # Should use Decimal for money
```

```go
// Integer boundary tests
func TestPagination(t *testing.T) {
    tests := []struct {
        name     string
        page     int
        pageSize int
        wantErr  bool
    }{
        {"page zero", 0, 10, true},
        {"negative page", -1, 10, true},
        {"page size zero", 1, 0, true},
        {"page size negative", 1, -1, true},
        {"page size at max", 1, MaxPageSize, false},
        {"page size exceeds max", 1, MaxPageSize + 1, true},
        {"very large page number", math.MaxInt32, 10, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := Paginate(tt.page, tt.pageSize)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| No empty/zero/negative tests for functions that accept collections or numbers | HIGH |
| No boundary tests for user-facing input limits (pagination, search, upload size) | HIGH |
| No overflow tests for financial calculations or quantity arithmetic | HIGH |
| Float equality used instead of approximate comparison for money | HIGH |
| Missing boundary tests for non-critical utility functions | MEDIUM |
| Thorough boundary value tests for all public API inputs | INFO (positive) |

---

## 2. Null / Nil / None Handling

Null reference errors are the most common runtime exception across all languages. Every function that accepts a reference type should be tested with null/nil/None input.

### Detection Patterns -- Missing Null Tests

```python
# Function under test
def format_user_name(user):
    return f"{user.first_name} {user.last_name}"

# Missing null tests:
def test_format_user_name_with_none():
    with pytest.raises(AttributeError):
        format_user_name(None)

def test_format_user_name_with_none_first_name():
    user = User(first_name=None, last_name="Smith")
    # What happens? TypeError? "None Smith"?
    with pytest.raises(TypeError):
        format_user_name(user)
```

```javascript
// Test null, undefined, and missing properties
test('formatUserName handles null user', () => {
  expect(() => formatUserName(null)).toThrow('User is required');
});

test('formatUserName handles undefined user', () => {
  expect(() => formatUserName(undefined)).toThrow('User is required');
});

test('formatUserName handles missing firstName', () => {
  expect(formatUserName({ lastName: 'Smith' })).toBe('Smith');
  // Or should it throw? The test documents the expected behavior
});
```

```go
func TestFormatUserName(t *testing.T) {
    t.Run("nil user returns error", func(t *testing.T) {
        _, err := FormatUserName(nil)
        assert.Error(t, err)
        assert.ErrorContains(t, err, "user is required")
    })

    t.Run("empty name fields", func(t *testing.T) {
        user := &User{FirstName: "", LastName: ""}
        result, err := FormatUserName(user)
        require.NoError(t, err)
        assert.Equal(t, "", result)  // Or should it error?
    })
}
```

```java
@Test
void formatUserName_nullUser_throwsNullPointerException() {
    assertThrows(NullPointerException.class, () -> {
        formatUserName(null);
    });
}

@Test
void formatUserName_nullFirstName_returnsLastNameOnly() {
    User user = new User(null, "Smith");
    assertEquals("Smith", formatUserName(user));
}

@Test
@DisplayName("Optional fields should not cause NPE when absent")
void formatUserName_optionalMiddleName_isNull() {
    User user = new User("Alice", "Smith");
    user.setMiddleName(null);
    assertEquals("Alice Smith", formatUserName(user));
}
```

### Null in Collections

```python
# Often overlooked: null elements within a collection
def test_process_users_with_none_in_list():
    users = [User("Alice"), None, User("Bob")]
    # What happens when the function iterates and hits None?
    with pytest.raises(TypeError):
        process_users(users)

def test_process_users_with_none_fields_in_objects():
    users = [User(name=None), User(name="Bob")]
    # The collection is not null, but elements have null fields
    result = process_users(users)
    assert len(result) == 1  # Skipped the invalid user? Or error?
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Public API functions with no null/nil/None input tests | HIGH |
| Functions accepting collections with no null-element tests | MEDIUM |
| Null handling behavior undocumented and untested | MEDIUM |
| Comprehensive null tests for all reference-type parameters | INFO (positive) |

---

## 3. Unicode and Special Characters

Text handling bugs are widespread and often invisible during development because developers test with ASCII-only inputs. Unicode, emoji, RTL text, and special characters must be tested explicitly.

### Critical Unicode Test Cases

| Category | Test Values | Why |
|----------|-------------|-----|
| Empty string | `""` | Different from null; often handled differently |
| Whitespace only | `" "`, `"\t"`, `"\n"` | May be treated as empty or valid depending on context |
| Leading/trailing whitespace | `" Alice "` | Trim behavior varies |
| Unicode letters | `"Renee"` (with accented e) | Normalization, encoding |
| CJK characters | `"田中太郎"` | Multi-byte encoding, string length |
| Emoji | `"Hello 👋🌍"` | Multi-codepoint characters, surrogate pairs |
| RTL text | `"مرحبا"` (Arabic) | Right-to-left rendering, bidirectional text |
| Null bytes | `"hello\x00world"` | C-string termination, security implications |
| SQL-significant | `"O'Brien"`, `"Robert'; DROP TABLE"` | Injection, escaping |
| HTML-significant | `"<script>alert('xss')</script>"` | XSS, escaping |
| Path-significant | `"../../etc/passwd"`, `"CON"`, `"NUL"` | Path traversal, Windows reserved names |
| Very long string | 10,000+ characters | Buffer overflow, performance, truncation |
| Zero-width characters | `"hello\u200Bworld"` (zero-width space) | Invisible characters, comparison issues |

### Detection Patterns -- Missing Unicode Tests

```python
# Function under test: user registration
def register_user(name, email):
    ...

# Missing tests:
def test_register_user_with_unicode_name():
    user = register_user("Renee Muller", "renee@test.com")
    assert user.name == "Renee Muller"

def test_register_user_with_cjk_name():
    user = register_user("田中太郎", "tanaka@test.com")
    assert user.name == "田中太郎"

def test_register_user_with_emoji_in_name():
    user = register_user("Alice 🚀", "alice@test.com")
    assert user.name == "Alice 🚀"

def test_register_user_with_html_in_name():
    user = register_user("<b>Alice</b>", "alice@test.com")
    assert user.name == "<b>Alice</b>"  # Stored literally, escaped on display
    # Or: assert user.name == "Alice" -- stripped tags
```

```javascript
test('search handles Unicode input', () => {
  const results = search('cafe\u0301');  // "cafe" + combining accent
  // Should match "cafe" (precomposed form)
  expect(results).toContainEqual(expect.objectContaining({ name: 'cafe' }));
});

test('string length with emoji', () => {
  // "👨‍👩‍👧‍👦" is a single visible emoji but multiple codepoints
  const familyEmoji = '👨‍👩‍👧‍👦';
  expect(familyEmoji.length).toBe(11);  // JavaScript string length!
  expect([...familyEmoji].length).toBe(7);  // Codepoints
  // If your function limits "character count," which count do you mean?
});
```

```go
func TestNormalization(t *testing.T) {
    // These two strings look identical but are different byte sequences
    precomposed := "cafe\u0301"  // 'e' + combining accent
    composed := "cafe"         // precomposed e-with-accent

    // Without normalization, these will not match
    assert.NotEqual(t, precomposed, composed, "raw bytes differ")

    // After NFC normalization, they should match
    normalized := norm.NFC.String(precomposed)
    assert.Equal(t, composed, normalized)
}

func TestStringTruncation(t *testing.T) {
    // Truncating multi-byte characters can produce invalid UTF-8
    input := "Hello 世界"
    truncated := SafeTruncate(input, 8)  // Must not split a multi-byte char
    assert.True(t, utf8.ValidString(truncated))
}
```

```java
@Test
void searchHandlesUnicode() {
    List<Result> results = searchService.search("cafe\u0301");
    // Should match "cafe" via Unicode normalization
    assertFalse(results.isEmpty(), "Normalized search should find 'cafe'");
}

@Test
void usernameValidation_rejectsZeroWidthCharacters() {
    assertThrows(ValidationException.class, () -> {
        userService.validateUsername("admin\u200B");  // "admin" + zero-width space
    });
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| User-facing text input with no Unicode tests | HIGH |
| Search/comparison functions with no normalization tests | HIGH |
| No tests for HTML/SQL-significant characters in user input | HIGH (security overlap) |
| String truncation without multi-byte safety tests | MEDIUM |
| ASCII-only test data throughout the entire suite | MEDIUM |
| Comprehensive Unicode and special character test coverage | INFO (positive) |

---

## 4. Large Input Testing

Functions should be tested with inputs significantly larger than typical usage to catch performance degradation, memory exhaustion, and algorithmic complexity issues.

### Detection Patterns -- Missing Large Input Tests

```python
# Function works fine with 10 items -- what about 1,000,000?
def test_sort_large_input():
    data = list(range(1_000_000, 0, -1))  # Worst case for some sorts
    result = custom_sort(data)
    assert result == sorted(data)

def test_csv_parser_large_file(tmp_path):
    # Generate a 100MB CSV
    csv_file = tmp_path / "large.csv"
    with open(csv_file, 'w') as f:
        f.write("name,value\n")
        for i in range(1_000_000):
            f.write(f"item_{i},{i * 1.5}\n")
    result = parse_csv(csv_file)
    assert len(result) == 1_000_000
```

```javascript
test('handles large JSON payload', () => {
  const largePayload = {
    items: Array.from({ length: 10000 }, (_, i) => ({
      id: i,
      name: `Item ${i}`,
      nested: { data: 'x'.repeat(100) },
    })),
  };
  const result = processPayload(largePayload);
  expect(result.processed).toBe(10000);
});

test('regex does not catastrophically backtrack', () => {
  const maliciousInput = 'a'.repeat(50) + '!';
  const start = Date.now();
  validateInput(maliciousInput);
  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(1000);  // Should not take more than 1 second
});
```

```go
func TestLargeSliceProcessing(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping large input test in short mode")
    }
    input := make([]int, 10_000_000)
    for i := range input {
        input[i] = rand.Intn(1000)
    }
    result := Process(input)
    assert.Equal(t, 10_000_000, len(result))
}

func BenchmarkProcess(b *testing.B) {
    input := make([]int, 10_000)
    for i := range input {
        input[i] = rand.Intn(1000)
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Process(input)
    }
}
```

```java
@Test
@Tag("slow")
void processLargeCollection() {
    List<Item> items = IntStream.range(0, 1_000_000)
        .mapToObj(i -> new Item("item_" + i, i * 1.5))
        .collect(Collectors.toList());

    long start = System.nanoTime();
    List<Result> results = service.process(items);
    long elapsed = Duration.ofNanos(System.nanoTime() - start).toMillis();

    assertEquals(1_000_000, results.size());
    assertTrue(elapsed < 5000, "Processing took " + elapsed + "ms, expected < 5000ms");
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| User-facing endpoints with no large input tests (DoS risk) | HIGH |
| Regex patterns with no backtracking tests | HIGH |
| Data processing functions with no performance/scale tests | MEDIUM |
| Well-tagged slow tests for large input scenarios | INFO (positive) |

---

## 5. Concurrent Access Scenarios

Concurrent access bugs are among the hardest to reproduce and the most damaging in production. Tests should specifically target shared state, race conditions, and concurrent modification.

### Detection Patterns -- Missing Concurrency Tests

```python
# Thread-safety test for a shared counter
import threading

def test_counter_thread_safety():
    counter = Counter()
    errors = []

    def increment_many():
        try:
            for _ in range(10_000):
                counter.increment()
        except Exception as e:
            errors.append(e)

    threads = [threading.Thread(target=increment_many) for _ in range(10)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert not errors
    assert counter.value == 100_000  # 10 threads * 10,000 increments
```

```go
// Go makes concurrency testing straightforward with goroutines and the race detector
func TestCacheThreadSafety(t *testing.T) {
    cache := NewCache()
    var wg sync.WaitGroup

    // Concurrent writes
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("key-%d", i), i)
        }(i)
    }

    // Concurrent reads
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            cache.Get(fmt.Sprintf("key-%d", i))
        }(i)
    }

    wg.Wait()
    // Run with: go test -race ./...
}
```

```java
@Test
void concurrentMapAccess_doesNotLoseUpdates() throws InterruptedException {
    ConcurrentMap<String, Integer> counters = new ConcurrentHashMap<>();
    int threadCount = 10;
    int incrementsPerThread = 10_000;
    CountDownLatch latch = new CountDownLatch(threadCount);
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            for (int j = 0; j < incrementsPerThread; j++) {
                counters.merge("counter", 1, Integer::sum);
            }
            latch.countDown();
        });
    }

    latch.await(30, TimeUnit.SECONDS);
    assertEquals(threadCount * incrementsPerThread, counters.get("counter"));
}
```

```javascript
// Node.js: test async concurrent access
test('concurrent database writes do not lose updates', async () => {
  const promises = Array.from({ length: 100 }, (_, i) =>
    updateCounter('test-counter', 1)  // Each increments by 1
  );
  await Promise.all(promises);

  const counter = await getCounter('test-counter');
  expect(counter.value).toBe(100);
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Shared mutable state (caches, counters, connection pools) with no concurrency tests | HIGH |
| Database operations with no concurrent access tests | HIGH |
| Go code with shared state never tested with `-race` flag | HIGH |
| Concurrent write tests present but no concurrent read+write tests | MEDIUM |
| Thorough concurrency tests with race detection enabled | INFO (positive) |

---

## 6. Error Path Testing

Error paths are the most undertested part of most codebases. Every error condition -- exceptions, validation failures, timeouts, resource exhaustion, partial failures -- should have a corresponding test.

### Exception / Error Return Testing

```python
# Every except/raise should have a test
def transfer_funds(from_account, to_account, amount):
    if amount <= 0:
        raise ValueError("Amount must be positive")
    if from_account.balance < amount:
        raise InsufficientFundsError(from_account, amount)
    try:
        from_account.debit(amount)
        to_account.credit(amount)
    except DatabaseError as e:
        from_account.rollback()
        raise TransferError("Database error during transfer") from e

# Required error path tests:
def test_transfer_negative_amount():
    with pytest.raises(ValueError, match="Amount must be positive"):
        transfer_funds(account_a, account_b, -10)

def test_transfer_zero_amount():
    with pytest.raises(ValueError, match="Amount must be positive"):
        transfer_funds(account_a, account_b, 0)

def test_transfer_insufficient_funds():
    account = Account(balance=50)
    with pytest.raises(InsufficientFundsError):
        transfer_funds(account, other_account, 100)

def test_transfer_database_error_triggers_rollback(mock_db):
    mock_db.side_effect = DatabaseError("Connection lost")
    with pytest.raises(TransferError, match="Database error"):
        transfer_funds(account_a, account_b, 50)
    assert account_a.balance == original_balance  # Verify rollback
```

### Timeout Testing

```python
@patch('app.client.requests.get')
def test_external_api_timeout(mock_get):
    mock_get.side_effect = requests.Timeout("Connection timed out")
    result = fetch_user_data(user_id=1)
    assert result.error == "Service temporarily unavailable"
    assert result.data is None
```

```go
func TestHTTPClientTimeout(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(5 * time.Second)  // Simulate slow server
    }))
    defer server.Close()

    client := NewClient(server.URL, WithTimeout(100*time.Millisecond))
    _, err := client.Get("/slow")
    assert.Error(t, err)
    assert.True(t, errors.Is(err, context.DeadlineExceeded) || os.IsTimeout(err))
}
```

```java
@Test
void httpClient_handlesTimeout() {
    wireMock.stubFor(get("/api/data")
        .willReturn(aResponse().withFixedDelay(5000)));  // 5 second delay

    ApiClient client = new ApiClient(wireMock.baseUrl(), Duration.ofMillis(100));
    assertThrows(TimeoutException.class, () -> client.getData());
}
```

### Resource Exhaustion Testing

```python
def test_handles_disk_full_gracefully(tmp_path, monkeypatch):
    def mock_write(*args, **kwargs):
        raise OSError(28, "No space left on device")

    monkeypatch.setattr("builtins.open", mock_write)
    result = save_report(data, tmp_path / "report.csv")
    assert result.success is False
    assert "disk" in result.error.lower()
```

```go
func TestDatabaseConnectionPoolExhaustion(t *testing.T) {
    db := setupTestDB(t, WithMaxConnections(2))

    // Acquire all connections
    conn1, _ := db.Acquire(context.Background())
    conn2, _ := db.Acquire(context.Background())

    // Third acquisition should timeout
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    _, err := db.Acquire(ctx)
    assert.Error(t, err)

    conn1.Release()
    conn2.Release()
}
```

### Partial Failure Testing

```python
# Batch operation where some items succeed and some fail
def test_batch_import_partial_failure():
    items = [
        {"name": "valid_1", "price": 10.00},
        {"name": "", "price": 5.00},            # Invalid: empty name
        {"name": "valid_2", "price": 20.00},
        {"name": "valid_3", "price": -1.00},     # Invalid: negative price
    ]
    result = batch_import(items)
    assert result.succeeded == 2
    assert result.failed == 2
    assert result.errors[1].field == "name"
    assert result.errors[3].field == "price"
```

```javascript
test('bulk email send handles partial failures', async () => {
  const recipients = [
    'valid@test.com',
    'invalid-email',
    'another@test.com',
    'also-invalid',
  ];

  const result = await sendBulkEmail(recipients, 'Hello');

  expect(result.sent).toBe(2);
  expect(result.failed).toBe(2);
  expect(result.errors).toHaveLength(2);
  expect(result.errors[0].recipient).toBe('invalid-email');
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Try/catch blocks with no corresponding error path test | HIGH |
| Network-calling code with no timeout test | HIGH |
| Batch operations with no partial failure test | HIGH |
| Resource-dependent code with no exhaustion test | MEDIUM |
| Database transaction code with no rollback test | HIGH |
| Comprehensive error path coverage with specific error assertions | INFO (positive) |

---

## 7. Security-Related Testing

While security testing is the domain of the security reviewer, the testing reviewer should flag missing tests for security-adjacent behavior: authentication boundaries, input sanitization, access control, and rate limiting.

### Authentication Boundary Tests

```python
# Test that unauthenticated access is properly rejected
def test_protected_endpoint_without_token():
    response = client.get("/api/admin/users")
    assert response.status_code == 401

def test_protected_endpoint_with_expired_token():
    token = generate_token(user, expires_in=-1)  # Already expired
    response = client.get("/api/admin/users", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 401

def test_protected_endpoint_with_invalid_signature():
    token = generate_token(user, secret="wrong-secret")
    response = client.get("/api/admin/users", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 401

def test_protected_endpoint_with_tampered_payload():
    token = generate_token(user)
    # Modify the payload without re-signing
    parts = token.split(".")
    parts[1] = base64_encode(json.dumps({"role": "admin", "sub": user.id}))
    tampered = ".".join(parts)
    response = client.get("/api/admin/users", headers={"Authorization": f"Bearer {tampered}"})
    assert response.status_code == 401
```

### Injection Vector Tests

```python
# SQL injection prevention test
def test_search_with_sql_injection_attempt():
    malicious_input = "'; DROP TABLE users; --"
    results = search_users(malicious_input)
    # Should return no results, NOT an error (which would indicate
    # the SQL was executed and failed)
    assert results == []
    # Verify the table still exists
    assert User.query.count() > 0
```

```javascript
test('API rejects HTML in text fields', async () => {
  const response = await request(app)
    .post('/api/comments')
    .send({ body: '<script>alert("xss")</script>' })
    .set('Authorization', `Bearer ${token}`);

  // Either reject the input or sanitize it
  if (response.status === 201) {
    expect(response.body.body).not.toContain('<script>');
  } else {
    expect(response.status).toBe(400);
  }
});
```

```go
func TestSQLInjectionPrevention(t *testing.T) {
    inputs := []string{
        "'; DROP TABLE users; --",
        "1 OR 1=1",
        "1; SELECT * FROM passwords",
        "1 UNION SELECT * FROM users",
    }
    for _, input := range inputs {
        t.Run(input, func(t *testing.T) {
            _, err := repo.FindByID(input)
            // Should return "not found" error, not a SQL syntax error
            assert.ErrorIs(t, err, ErrNotFound)
        })
    }
}
```

### Access Control Tests

```java
@Test
void regularUserCannotAccessAdminEndpoint() {
    String userToken = getTokenForRole("user");
    Response response = given()
        .header("Authorization", "Bearer " + userToken)
        .get("/api/admin/config");
    assertEquals(403, response.statusCode());
}

@Test
void userCannotAccessAnotherUsersData() {
    String aliceToken = getTokenForUser("alice");
    Response response = given()
        .header("Authorization", "Bearer " + aliceToken)
        .get("/api/users/bob/profile");
    assertEquals(403, response.statusCode());
}

@Test
void deletedUserTokenIsRejected() {
    String token = getTokenForUser("charlie");
    deleteUser("charlie");
    Response response = given()
        .header("Authorization", "Bearer " + token)
        .get("/api/profile");
    assertEquals(401, response.statusCode());
}
```

### Rate Limiting Tests

```python
def test_login_rate_limiting():
    for i in range(10):
        response = client.post("/api/login", json={
            "username": "admin",
            "password": "wrong-password",
        })
    # After 10 failed attempts, should be rate limited
    response = client.post("/api/login", json={
        "username": "admin",
        "password": "correct-password",
    })
    assert response.status_code == 429
    assert "retry-after" in response.headers
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| No tests for unauthenticated access to protected endpoints | CRITICAL |
| No tests for cross-user data access (horizontal privilege) | CRITICAL |
| No tests for SQL/command injection prevention | HIGH |
| No tests for expired/invalid token rejection | HIGH |
| No rate limiting tests for authentication endpoints | HIGH |
| No XSS prevention tests for user-generated content | HIGH |
| Comprehensive security boundary tests | INFO (positive) |

---

## 8. Race Conditions

Race conditions are timing-dependent bugs that occur when multiple operations access shared state without proper synchronization. They are notoriously difficult to reproduce but must be tested.

### Common Race Condition Patterns

**Double-submit / idempotency**:

```python
import concurrent.futures

def test_double_submit_payment():
    """Submitting the same payment twice concurrently should not charge twice."""
    order = create_order(amount=100.00)
    idempotency_key = "unique-key-123"

    with concurrent.futures.ThreadPoolExecutor(max_workers=2) as executor:
        futures = [
            executor.submit(process_payment, order.id, idempotency_key),
            executor.submit(process_payment, order.id, idempotency_key),
        ]
        results = [f.result() for f in concurrent.futures.as_completed(futures)]

    successes = [r for r in results if r.status == "success"]
    duplicates = [r for r in results if r.status == "duplicate"]
    assert len(successes) == 1
    assert len(duplicates) == 1
    assert get_total_charges(order.id) == 100.00  # Charged exactly once
```

**Check-then-act (TOCTOU)**:

```go
// Test for time-of-check-to-time-of-use race
func TestInventoryReservation_ConcurrentPurchase(t *testing.T) {
    // One item in stock, two concurrent purchase attempts
    product := createProduct(t, stock: 1)

    var wg sync.WaitGroup
    results := make([]error, 2)

    for i := 0; i < 2; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            results[idx] = purchaseProduct(product.ID, quantity: 1)
        }(i)
    }
    wg.Wait()

    // Exactly one should succeed, one should fail with "out of stock"
    successCount := 0
    for _, err := range results {
        if err == nil {
            successCount++
        }
    }
    assert.Equal(t, 1, successCount, "exactly one purchase should succeed")
    assert.Equal(t, 0, getStock(product.ID), "stock should be zero")
}
```

**Read-modify-write race**:

```java
@Test
void concurrentBalanceUpdates_maintainConsistency() throws Exception {
    Account account = createAccount(1000.00);
    int threadCount = 10;
    double withdrawalAmount = 100.00;
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        new Thread(() -> {
            try {
                startLatch.await();  // All threads start simultaneously
                accountService.withdraw(account.getId(), withdrawalAmount);
            } catch (Exception e) {
                // Expected for some threads
            } finally {
                doneLatch.countDown();
            }
        }).start();
    }

    startLatch.countDown();  // Release all threads at once
    doneLatch.await(30, TimeUnit.SECONDS);

    Account updated = accountService.getAccount(account.getId());
    assertTrue(updated.getBalance() >= 0, "Balance should never go negative");
    assertEquals(0, updated.getBalance() % withdrawalAmount,
        "Balance should be a multiple of withdrawal amount");
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Payment/financial operations with no double-submit test | CRITICAL |
| Inventory/stock operations with no concurrent purchase test | HIGH |
| Shared counter/balance with no concurrent update test | HIGH |
| Cache operations with no concurrent read/write test | MEDIUM |
| Comprehensive race condition tests with concurrent execution | INFO (positive) |

---

## 9. Timezone and Locale Sensitivity

Date/time and locale-dependent code is a persistent source of bugs that often only manifest in production with real users in different timezones and locales.

### Timezone Edge Cases

```python
import pytz
from datetime import datetime, timezone

def test_event_spans_dst_transition():
    """Events created before DST change should display correctly after."""
    eastern = pytz.timezone("US/Eastern")
    # March 10, 2024: DST spring forward at 2:00 AM
    before_dst = eastern.localize(datetime(2024, 3, 10, 1, 30))
    after_dst = eastern.localize(datetime(2024, 3, 10, 3, 30))
    duration = after_dst - before_dst
    assert duration.total_seconds() == 3600  # Only 1 hour, not 2

def test_midnight_in_different_timezones():
    """A date can be different depending on timezone."""
    utc_time = datetime(2024, 1, 15, 23, 0, tzinfo=timezone.utc)
    # In UTC it is Jan 15; in UTC+2 it is already Jan 16
    eastern = utc_time.astimezone(pytz.timezone("US/Eastern"))
    tokyo = utc_time.astimezone(pytz.timezone("Asia/Tokyo"))
    assert eastern.day == 15
    assert tokyo.day == 16  # Already the next day in Tokyo

def test_naive_datetime_comparison():
    """Comparing naive and aware datetimes should raise an error."""
    naive = datetime(2024, 1, 15, 12, 0)
    aware = datetime(2024, 1, 15, 12, 0, tzinfo=timezone.utc)
    with pytest.raises(TypeError):
        naive < aware
```

```javascript
test('date formatting respects user timezone', () => {
  const utcDate = new Date('2024-01-15T23:00:00Z');

  const nyFormatted = formatDate(utcDate, 'America/New_York');
  expect(nyFormatted).toBe('January 15, 2024');

  const tokyoFormatted = formatDate(utcDate, 'Asia/Tokyo');
  expect(tokyoFormatted).toBe('January 16, 2024');  // Next day in Tokyo
});

test('handles invalid timezone gracefully', () => {
  const date = new Date('2024-01-15T12:00:00Z');
  expect(() => formatDate(date, 'Invalid/Timezone')).toThrow('Invalid timezone');
});
```

```go
func TestTimezoneHandling(t *testing.T) {
    t.Run("DST transition", func(t *testing.T) {
        loc, _ := time.LoadLocation("America/New_York")
        // 2:30 AM does not exist on DST spring forward day
        ambiguous := time.Date(2024, 3, 10, 2, 30, 0, 0, loc)
        // Go resolves this -- verify the behavior your code expects
        assert.Equal(t, 3, ambiguous.Hour(), "2:30 AM should resolve to 3:30 AM EST->EDT")
    })

    t.Run("end of day in UTC vs local", func(t *testing.T) {
        utcEndOfDay := time.Date(2024, 1, 15, 23, 59, 59, 0, time.UTC)
        tokyo, _ := time.LoadLocation("Asia/Tokyo")
        inTokyo := utcEndOfDay.In(tokyo)
        assert.Equal(t, 16, inTokyo.Day())
    })
}
```

### Locale Edge Cases

```java
@Test
void currencyFormatting_respectsLocale() {
    BigDecimal amount = new BigDecimal("1234.56");

    String usFormat = formatCurrency(amount, Locale.US);
    assertEquals("$1,234.56", usFormat);

    String germanFormat = formatCurrency(amount, Locale.GERMANY);
    assertEquals("1.234,56 EUR", germanFormat);  // Reversed separators!

    String japaneseFormat = formatCurrency(amount, Locale.JAPAN);
    assertEquals("JPY1,235", japaneseFormat);  // No decimal places for JPY
}

@Test
void stringComparison_respectsLocale() {
    // Turkish locale: uppercase of "i" is "I" (with a dot), not "I"
    Locale turkish = new Locale("tr", "TR");
    assertEquals("I", "i".toUpperCase(turkish));  // Different from English!
    // This breaks code like: if (input.toUpperCase().equals("FILE"))
}
```

```python
def test_date_parsing_locale_format():
    """Date format varies by locale: MM/DD/YYYY vs DD/MM/YYYY."""
    # Is "01/02/2024" January 2nd or February 1st?
    us_date = parse_date("01/02/2024", locale="en_US")
    assert us_date.month == 1
    assert us_date.day == 2

    uk_date = parse_date("01/02/2024", locale="en_GB")
    assert uk_date.month == 2
    assert uk_date.day == 1
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Date/time code with no timezone tests | HIGH |
| Financial/reporting code with no DST transition tests | HIGH |
| User-facing date formatting with no locale tests | MEDIUM |
| String comparison using default locale in internationalized app | MEDIUM |
| Currency formatting with no locale-specific tests | MEDIUM |
| Comprehensive timezone and locale test coverage | INFO (positive) |

---

## 10. File System Edge Cases

File system operations have many failure modes that are rarely tested: permission errors, full disks, long paths, symlinks, concurrent access, and platform differences.

### Permission Errors

```python
def test_write_to_readonly_directory(tmp_path):
    readonly_dir = tmp_path / "readonly"
    readonly_dir.mkdir()
    readonly_dir.chmod(0o444)

    with pytest.raises(PermissionError):
        save_file(readonly_dir / "output.txt", "data")

    # Cleanup: restore permissions so pytest can clean up tmp_path
    readonly_dir.chmod(0o755)

def test_read_file_without_permission(tmp_path):
    secret = tmp_path / "secret.txt"
    secret.write_text("sensitive data")
    secret.chmod(0o000)

    with pytest.raises(PermissionError):
        read_file(secret)

    secret.chmod(0o644)
```

```go
func TestWriteToReadOnlyDir(t *testing.T) {
    dir := t.TempDir()
    err := os.Chmod(dir, 0o444)
    require.NoError(t, err)
    defer os.Chmod(dir, 0o755)

    err = WriteFile(filepath.Join(dir, "test.txt"), []byte("data"))
    assert.Error(t, err)
    assert.True(t, os.IsPermission(err))
}
```

### Full Disk Simulation

```python
def test_handles_disk_full(monkeypatch):
    original_open = open

    call_count = 0
    def mock_open(*args, **kwargs):
        nonlocal call_count
        f = original_open(*args, **kwargs)
        original_write = f.write
        def failing_write(data):
            nonlocal call_count
            call_count += 1
            if call_count > 10:
                raise OSError(28, "No space left on device")
            return original_write(data)
        f.write = failing_write
        return f

    monkeypatch.setattr("builtins.open", mock_open)
    result = export_data(large_dataset)
    assert result.success is False
    assert "space" in result.error.lower()
```

### Long Paths

```python
def test_handles_very_long_path(tmp_path):
    # Windows MAX_PATH is 260 characters; Linux typically allows 4096
    long_name = "a" * 200
    deep_path = tmp_path
    for _ in range(5):
        deep_path = deep_path / long_name

    # Should either succeed or raise a clear error, not crash
    try:
        deep_path.mkdir(parents=True)
        save_file(deep_path / "test.txt", "data")
    except OSError as e:
        assert "name too long" in str(e).lower() or "path" in str(e).lower()
```

```javascript
test('handles long file paths gracefully', () => {
  const longPath = 'a'.repeat(300) + '.txt';
  expect(() => readConfigFile(longPath)).toThrow(/path|name too long/i);
});
```

### Symlinks and Special Files

```python
def test_does_not_follow_symlinks_outside_directory(tmp_path):
    """Prevent path traversal via symlink."""
    allowed_dir = tmp_path / "uploads"
    allowed_dir.mkdir()

    # Create symlink pointing outside the allowed directory
    secret = tmp_path / "secret.txt"
    secret.write_text("sensitive")
    symlink = allowed_dir / "link.txt"
    symlink.symlink_to(secret)

    with pytest.raises(SecurityError):
        read_uploaded_file(symlink)  # Should detect symlink escape
```

```go
func TestResolvePathPreventsTraversal(t *testing.T) {
    baseDir := t.TempDir()

    // Create a file outside the base directory
    outsideFile := filepath.Join(os.TempDir(), "secret.txt")
    os.WriteFile(outsideFile, []byte("secret"), 0644)
    defer os.Remove(outsideFile)

    // Attempt traversal
    maliciousPath := filepath.Join(baseDir, "..", "..", "..", outsideFile)
    _, err := SafeResolvePath(baseDir, maliciousPath)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "outside base directory")
}
```

### Concurrent File Access

```java
@Test
void concurrentFileWrites_doNotCorruptData() throws Exception {
    Path testFile = tempDir.resolve("concurrent.txt");
    int writerCount = 10;
    CountDownLatch latch = new CountDownLatch(writerCount);

    for (int i = 0; i < writerCount; i++) {
        final int writerId = i;
        new Thread(() -> {
            try {
                appendToFile(testFile, "Writer " + writerId + "\n");
            } finally {
                latch.countDown();
            }
        }).start();
    }

    latch.await(10, TimeUnit.SECONDS);

    List<String> lines = Files.readAllLines(testFile);
    assertEquals(writerCount, lines.size(), "All writes should be present");
    // Each line should be complete (not interleaved)
    for (String line : lines) {
        assertTrue(line.matches("Writer \\d+"), "Line should not be corrupted: " + line);
    }
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| File upload/download with no symlink traversal test | HIGH |
| File operations with no permission error test | MEDIUM |
| No long path handling test for user-provided filenames | MEDIUM |
| File writing with no disk-full or I/O error test | MEDIUM |
| Concurrent file access with no corruption test | MEDIUM |
| No platform-specific path tests (Windows vs. Unix separators) | LOW |
| Comprehensive file system edge case coverage | INFO (positive) |

---

## Quick-Reference Summary

| Check | Key Boundary / Scenario | Severity |
|-------|------------------------|----------|
| No empty/zero/single-element tests | Collections and numbers | HIGH |
| No MAX_INT overflow tests | Financial/quantity arithmetic | HIGH |
| Float equality for money (no approx) | Precision errors | HIGH |
| No null/nil/None input tests | Public API functions | HIGH |
| No Unicode/special character tests | User-facing text input | HIGH |
| No large input tests | User-facing endpoints (DoS) | HIGH |
| No regex backtracking tests | Pattern matching on user input | HIGH |
| No concurrent access tests | Shared mutable state | HIGH |
| No error path tests for try/catch blocks | Exception handling | HIGH |
| No timeout tests for network calls | External dependencies | HIGH |
| No partial failure tests for batch ops | Batch processing | HIGH |
| No unauthenticated access tests | Protected endpoints | CRITICAL |
| No cross-user access tests | Authorization boundaries | CRITICAL |
| No SQL injection prevention tests | Database-backed search | HIGH |
| No double-submit / idempotency tests | Payment processing | CRITICAL |
| No timezone tests for date/time code | Scheduling, reporting | HIGH |
| No DST transition tests | Financial, scheduling | HIGH |
| No locale tests for formatting | Internationalized apps | MEDIUM |
| No file permission error tests | File I/O operations | MEDIUM |
| No path traversal tests | File upload/download | HIGH |
| No symlink escape tests | File serving | HIGH |
| Comprehensive edge case coverage | All categories | INFO (positive) |
