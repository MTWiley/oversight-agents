# Test Quality Reference

This reference provides detailed criteria for evaluating the quality of test code itself -- structure, naming, isolation, assertions, mocking, data management, and maintainability. A test suite with high coverage but poor test quality gives false confidence. Tests that are unclear, brittle, or interdependent become a maintenance burden rather than a safety net.

The `testing-reviewer` agent uses this file as a lookup when evaluating test quality. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Arrange-Act-Assert Structure

The Arrange-Act-Assert (AAA) pattern is the foundational structure for readable, maintainable tests. Each test should have three clearly delineated phases.

**Definition**:
- **Arrange**: Set up preconditions and inputs
- **Act**: Execute the behavior under test (ideally a single call)
- **Assert**: Verify the expected outcome

### Detection Patterns -- Violations

**Interleaved arrange/act/assert**:

```python
# Bad: arrange, act, and assert mixed together
def test_order_processing():
    order = Order(items=[Item("widget", 10.00)])
    assert order.total == 10.00                    # Asserting during arrange
    order.apply_discount(0.1)
    assert order.total == 9.00
    order.add_item(Item("gadget", 20.00))
    assert order.total == 27.00                    # Multiple acts and asserts interleaved
    order.finalize()
    assert order.status == "finalized"

# Good: separate tests for each behavior
def test_order_total_reflects_items():
    order = Order(items=[Item("widget", 10.00)])
    total = order.total
    assert total == 10.00

def test_discount_reduces_total():
    order = Order(items=[Item("widget", 10.00)])
    order.apply_discount(0.1)
    assert order.total == 9.00

def test_finalize_sets_status():
    order = Order(items=[Item("widget", 10.00)])
    order.finalize()
    assert order.status == "finalized"
```

```javascript
// Bad: multiple acts with assertions between them
test('user registration flow', () => {
  const user = createUser({ name: 'Alice', email: 'alice@test.com' });
  expect(user.id).toBeDefined();

  activateUser(user.id);
  expect(user.isActive).toBe(true);

  const profile = getProfile(user.id);
  expect(profile.name).toBe('Alice');
});

// Good: one act per test
test('createUser assigns an ID', () => {
  const user = createUser({ name: 'Alice', email: 'alice@test.com' });
  expect(user.id).toBeDefined();
});

test('activateUser sets user as active', () => {
  const user = createUser({ name: 'Alice', email: 'alice@test.com' });
  activateUser(user.id);
  expect(user.isActive).toBe(true);
});
```

```go
// Bad: multiple actions in one test with no clear separation
func TestUserService(t *testing.T) {
    svc := NewUserService(mockDB)
    user, err := svc.Create("Alice", "alice@test.com")
    if err != nil {
        t.Fatal(err)
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID")
    }
    err = svc.Activate(user.ID)
    if err != nil {
        t.Fatal(err)
    }
    got, _ := svc.Get(user.ID)
    if !got.Active {
        t.Error("expected active user")
    }
}

// Good: use subtests to separate concerns
func TestUserService(t *testing.T) {
    t.Run("Create assigns an ID", func(t *testing.T) {
        svc := NewUserService(mockDB)
        user, err := svc.Create("Alice", "alice@test.com")
        require.NoError(t, err)
        assert.NotZero(t, user.ID)
    })

    t.Run("Activate sets user as active", func(t *testing.T) {
        svc := NewUserService(mockDB)
        user, _ := svc.Create("Alice", "alice@test.com")
        err := svc.Activate(user.ID)
        require.NoError(t, err)
        got, _ := svc.Get(user.ID)
        assert.True(t, got.Active)
    })
}
```

```java
// Bad: multiple acts
@Test
void testOrderLifecycle() {
    Order order = new Order();
    order.addItem(new Item("widget", 10.00));
    assertEquals(10.00, order.getTotal());          // Assert after first act
    order.applyDiscount(0.1);
    assertEquals(9.00, order.getTotal());            // Assert after second act
    order.submit();
    assertEquals(OrderStatus.SUBMITTED, order.getStatus());  // Assert after third act
}

// Good: one behavior per test
@Test
void addItem_increasesTotal() {
    Order order = new Order();
    order.addItem(new Item("widget", 10.00));
    assertEquals(10.00, order.getTotal());
}

@Test
void applyDiscount_reducesTotal() {
    Order order = new Order();
    order.addItem(new Item("widget", 10.00));
    order.applyDiscount(0.1);
    assertEquals(9.00, order.getTotal());
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tests with >3 interleaved act-assert cycles (integration-test-in-disguise) | MEDIUM |
| Most tests follow AAA but a few mix phases | LOW |
| Tests with clear Arrange/Act/Assert comments or separation | INFO (positive) |

---

## 2. Single Assertion Focus

Each test should verify one logical concept. Multiple assertions are acceptable when they verify different aspects of the same result (e.g., checking both the status code and body of an HTTP response), but should not test multiple behaviors.

### Heuristic

- **One assertion**: Ideal. Clear failure message, obvious what broke.
- **2-5 related assertions on the same result**: Acceptable. All verify the same action's outcome.
- **>5 assertions or assertions on different actions**: Likely testing too much. Split into separate tests.

### Detection Patterns

```python
# Bad: testing multiple behaviors in one test
def test_user_service():
    # Tests creation
    user = service.create_user("Alice", "alice@test.com")
    assert user.name == "Alice"
    assert user.email == "alice@test.com"

    # Tests update (separate behavior)
    service.update_user(user.id, name="Bob")
    updated = service.get_user(user.id)
    assert updated.name == "Bob"

    # Tests deletion (yet another behavior)
    service.delete_user(user.id)
    assert service.get_user(user.id) is None

# Good: multiple assertions on the same result are fine
def test_create_user_returns_complete_user():
    user = service.create_user("Alice", "alice@test.com")
    assert user.name == "Alice"
    assert user.email == "alice@test.com"
    assert user.id is not None
    assert user.created_at is not None
```

```javascript
// Good: multiple expects verifying the same response
test('GET /api/users/:id returns user details', async () => {
  const response = await request(app).get('/api/users/1');

  expect(response.status).toBe(200);
  expect(response.body.name).toBe('Alice');
  expect(response.body.email).toBe('alice@test.com');
  expect(response.headers['content-type']).toMatch(/json/);
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tests routinely verify 3+ separate behaviors (create, update, delete in one test) | MEDIUM |
| Occasional multi-behavior test in a generally well-structured suite | LOW |
| Tests with focused assertions on a single behavior | INFO (positive) |

---

## 3. Test Naming Conventions

Test names are documentation. A well-named test communicates what behavior is being verified and what the expected outcome is, without reading the test body.

### Language-Specific Conventions

**Python (pytest)**:

Pattern: `test_<unit>_<scenario>_<expected_result>` or `test_<behavior_description>`

```python
# Good: descriptive names
def test_calculate_discount_with_premium_membership_returns_twenty_percent():
    ...

def test_calculate_discount_with_expired_coupon_raises_value_error():
    ...

def test_parse_csv_with_empty_file_returns_empty_list():
    ...

# Bad: vague names
def test_discount():
    ...

def test_it_works():
    ...

def test_case_1():
    ...
```

**JavaScript/TypeScript (Jest, Mocha, Vitest)**:

Pattern: `describe('<Unit>') + it('should <behavior> when <condition>')` or `test('<unit> <behavior> when <condition>')`

```javascript
// Good: nested describe/it with clear behavior description
describe('ShoppingCart', () => {
  describe('addItem', () => {
    it('should increase the item count by one', () => { ... });
    it('should update the total price', () => { ... });
    it('should throw when item is out of stock', () => { ... });
  });

  describe('removeItem', () => {
    it('should decrease the item count by one', () => { ... });
    it('should not allow negative quantities', () => { ... });
  });
});

// Bad: meaningless names
test('test 1', () => { ... });
test('cart works', () => { ... });
test('addItem', () => { ... });  // What about addItem?
```

**Go**:

Pattern: `Test<Unit>_<Scenario>` or `Test<Unit>/<Scenario>` (subtests)

```go
// Good: subtests with clear scenario descriptions
func TestParseConfig(t *testing.T) {
    t.Run("valid YAML returns parsed config", func(t *testing.T) { ... })
    t.Run("empty file returns default config", func(t *testing.T) { ... })
    t.Run("invalid YAML returns parse error", func(t *testing.T) { ... })
    t.Run("missing required field returns validation error", func(t *testing.T) { ... })
}

// Good: table-driven tests with descriptive names
func TestCalculateTax(t *testing.T) {
    tests := []struct {
        name     string
        amount   float64
        region   string
        expected float64
    }{
        {"US standard rate", 100.0, "US", 8.0},
        {"EU VAT rate", 100.0, "EU", 20.0},
        {"zero amount", 0.0, "US", 0.0},
        {"tax-exempt region", 100.0, "EXEMPT", 0.0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := CalculateTax(tt.amount, tt.region)
            assert.Equal(t, tt.expected, got)
        })
    }
}

// Bad: no subtests, unclear test name
func TestTax(t *testing.T) {
    if CalculateTax(100, "US") != 8.0 {
        t.Fail()
    }
}
```

**Java (JUnit 5)**:

Pattern: `<methodName>_<scenario>_<expectedResult>` or `@DisplayName` with readable description

```java
// Good: method name with scenario and expectation
@Test
void calculateDiscount_premiumMember_returnsTwentyPercent() { ... }

@Test
void calculateDiscount_expiredCoupon_throwsIllegalArgumentException() { ... }

// Good: @DisplayName for readable test reports
@DisplayName("Should return 20% discount for premium members")
@Test
void premiumMemberDiscount() { ... }

// Bad: nondescript names
@Test
void test1() { ... }

@Test
void testCalculate() { ... }
```

**Ruby (RSpec)**:

Pattern: `describe '<Class>' + context 'when <condition>' + it '<expected behavior>'`

```ruby
# Good: nested context with clear descriptions
RSpec.describe ShoppingCart do
  describe '#add_item' do
    context 'when item is in stock' do
      it 'increases the item count' do ... end
      it 'updates the total price' do ... end
    end

    context 'when item is out of stock' do
      it 'raises an OutOfStockError' do ... end
    end
  end
end

# Bad: flat, vague descriptions
RSpec.describe ShoppingCart do
  it 'works' do ... end
  it 'does stuff' do ... end
end
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Most tests have nondescript names (test1, testIt, test_works) | MEDIUM |
| Inconsistent naming conventions within the same test suite | LOW |
| Tests follow consistent, descriptive naming conventions | INFO (positive) |

---

## 4. Test Independence and Isolation

Tests must not depend on execution order or shared mutable state. Each test should set up its own preconditions and clean up after itself. A test that passes only when run after another test is a hidden time bomb.

### Detection Patterns -- Shared State Violations

```python
# Bad: tests share mutable state through a module-level variable
user_id = None

def test_create_user():
    global user_id
    user_id = service.create_user("Alice").id
    assert user_id is not None

def test_get_user():
    # Depends on test_create_user running first
    user = service.get_user(user_id)
    assert user.name == "Alice"

# Good: each test creates its own state
def test_create_user():
    user = service.create_user("Alice")
    assert user.id is not None

def test_get_user():
    created = service.create_user("Alice")
    user = service.get_user(created.id)
    assert user.name == "Alice"
```

```javascript
// Bad: tests modify shared state
let testUser;

beforeAll(async () => {
  testUser = await createUser({ name: 'Alice' });
});

test('update user name', async () => {
  await updateUser(testUser.id, { name: 'Bob' });
  // testUser is now Bob -- next test sees modified state
});

test('get user returns original name', async () => {
  const user = await getUser(testUser.id);
  expect(user.name).toBe('Alice');  // FAILS: previous test changed it to Bob
});

// Good: each test gets a fresh user
test('update user name', async () => {
  const user = await createUser({ name: 'Alice' });
  await updateUser(user.id, { name: 'Bob' });
  const updated = await getUser(user.id);
  expect(updated.name).toBe('Bob');
});
```

```go
// Bad: package-level variable modified by tests
var testDB *sql.DB

func TestMain(m *testing.M) {
    testDB, _ = sql.Open("postgres", "...")
    os.Exit(m.Run())
}

func TestCreateUser(t *testing.T) {
    testDB.Exec("INSERT INTO users (name) VALUES ('Alice')")
    // Leaves data in testDB -- affects other tests
}

// Good: each test uses a transaction that rolls back
func TestCreateUser(t *testing.T) {
    tx, _ := testDB.Begin()
    defer tx.Rollback()

    _, err := tx.Exec("INSERT INTO users (name) VALUES ('Alice')")
    require.NoError(t, err)

    var name string
    tx.QueryRow("SELECT name FROM users WHERE name = 'Alice'").Scan(&name)
    assert.Equal(t, "Alice", name)
}
```

```java
// Bad: static field shared across tests
class UserServiceTest {
    static User sharedUser;

    @BeforeAll
    static void setup() {
        sharedUser = new User("Alice");
    }

    @Test
    void updateName() {
        sharedUser.setName("Bob");  // Mutates shared state
        assertEquals("Bob", sharedUser.getName());
    }

    @Test
    void originalName() {
        assertEquals("Alice", sharedUser.getName());  // May fail depending on order
    }
}

// Good: fresh instance per test
class UserServiceTest {
    private User user;

    @BeforeEach
    void setup() {
        user = new User("Alice");  // Fresh instance for each test
    }

    @Test
    void updateName() {
        user.setName("Bob");
        assertEquals("Bob", user.getName());
    }
}
```

### Detection Signals for Order Dependency

| Signal | What It Means |
|--------|---------------|
| Tests pass individually but fail when run together | Shared state pollution |
| Tests pass in one order but fail in another (`pytest --randomly`) | Order dependency |
| `global` keyword or module-level mutation in test files | Shared mutable state |
| `@BeforeAll`/`beforeAll` that creates mutable state not reset by `@BeforeEach`/`beforeEach` | Leak between tests |
| Test file has no setup/teardown but depends on database state | Implicit dependency on prior tests or fixtures |

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tests that fail when run in isolation or random order | HIGH |
| Mutable global/static state shared across tests | HIGH |
| Tests modifying shared fixtures without reset | MEDIUM |
| Tests using `beforeAll` for mutable state that `beforeEach` would be safer for | MEDIUM |
| Tests fully independent and runnable in any order | INFO (positive) |

---

## 5. Assertion Quality and Specificity

Assertions should be as specific as possible about the expected outcome. Vague assertions that pass for the wrong reasons give false confidence.

### Detection Patterns -- Weak Assertions

```python
# Bad: overly broad assertions
def test_get_user():
    user = service.get_user(1)
    assert user is not None          # Passes even if user has wrong data
    assert isinstance(user, dict)    # Passes even if dict is empty

# Good: specific assertions
def test_get_user():
    user = service.get_user(1)
    assert user["name"] == "Alice"
    assert user["email"] == "alice@example.com"
    assert user["role"] == "admin"
```

```javascript
// Bad: truthiness check
test('fetchUsers returns data', async () => {
  const users = await fetchUsers();
  expect(users).toBeTruthy();  // Passes for [], {}, 1, "anything"
});

// Good: specific shape and content
test('fetchUsers returns array of user objects', async () => {
  const users = await fetchUsers();
  expect(users).toHaveLength(3);
  expect(users[0]).toMatchObject({
    id: expect.any(Number),
    name: expect.any(String),
    email: expect.stringMatching(/@/),
  });
});
```

```go
// Bad: only checking error is nil
func TestCreateUser(t *testing.T) {
    user, err := service.CreateUser("Alice", "alice@test.com")
    if err != nil {
        t.Fatal(err)
    }
    // No verification of the returned user
}

// Good: verify the result
func TestCreateUser(t *testing.T) {
    user, err := service.CreateUser("Alice", "alice@test.com")
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, "alice@test.com", user.Email)
    assert.NotZero(t, user.ID)
    assert.WithinDuration(t, time.Now(), user.CreatedAt, time.Second)
}
```

```java
// Bad: boolean assertion losing context
@Test
void testUserCreation() {
    User user = service.createUser("Alice");
    assertTrue(user != null);  // Failure message: "expected true"
    assertTrue(user.getName().equals("Alice"));  // Failure message: "expected true"
}

// Good: specific assertions with meaningful failure messages
@Test
void createUser_returnsUserWithCorrectName() {
    User user = service.createUser("Alice");
    assertNotNull(user, "createUser should return non-null user");
    assertEquals("Alice", user.getName());
    assertNotNull(user.getId(), "created user should have an ID");
}
```

### Exception Assertion Quality

```python
# Bad: only checking that an exception is raised
def test_invalid_email():
    with pytest.raises(ValueError):
        validate_email("not-an-email")

# Good: checking exception type AND message
def test_invalid_email_includes_value_in_message():
    with pytest.raises(ValueError, match="Invalid email format: not-an-email"):
        validate_email("not-an-email")
```

```javascript
// Bad: just checking it throws
test('rejects invalid email', () => {
  expect(() => validateEmail('bad')).toThrow();
});

// Good: checking the error message
test('rejects invalid email with descriptive error', () => {
  expect(() => validateEmail('bad')).toThrow('Invalid email format: bad');
});
```

```java
// Good: assertThrows with message verification
@Test
void invalidEmail_throwsWithDescriptiveMessage() {
    IllegalArgumentException ex = assertThrows(
        IllegalArgumentException.class,
        () -> validateEmail("bad")
    );
    assertEquals("Invalid email format: bad", ex.getMessage());
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tests with no assertions at all (coverage-only tests) | HIGH |
| Tests using only truthiness/nullability checks (toBeTruthy, is not None) | MEDIUM |
| Exception tests that only check the type, not the message | LOW |
| Tests with specific, descriptive assertions | INFO (positive) |

---

## 6. Mock/Stub Quality and Realism

Mocks and stubs are essential for isolation, but unrealistic mocks create tests that pass while production fails. Mocks should reflect the actual behavior of the dependency they replace.

### Detection Patterns -- Unrealistic Mocks

```python
# Bad: mock always succeeds -- never tests error paths
@patch('app.services.payment_gateway.charge')
def test_process_payment(mock_charge):
    mock_charge.return_value = {"status": "success", "id": "txn_123"}
    result = process_payment(100.00, "usd", "card_123")
    assert result.status == "success"
    # Where is the test for declined cards, network errors, timeouts?

# Good: realistic mock that covers multiple scenarios
@patch('app.services.payment_gateway.charge')
def test_process_payment_declined(mock_charge):
    mock_charge.return_value = {
        "status": "declined",
        "decline_code": "insufficient_funds",
        "message": "Your card has insufficient funds.",
    }
    result = process_payment(100.00, "usd", "card_123")
    assert result.status == "declined"
    assert result.decline_reason == "insufficient_funds"

@patch('app.services.payment_gateway.charge')
def test_process_payment_gateway_timeout(mock_charge):
    mock_charge.side_effect = requests.Timeout("Connection timed out")
    with pytest.raises(PaymentError, match="Gateway timeout"):
        process_payment(100.00, "usd", "card_123")
```

```javascript
// Bad: mock returns a shape that the real API does not
jest.mock('./api');
api.getUser.mockResolvedValue({ name: 'Alice' });
// Real API returns: { data: { user: { name: 'Alice', id: 1 } }, status: 200 }
// This test passes but production code would break accessing response.data.user

// Good: mock matches the real API contract
api.getUser.mockResolvedValue({
  data: { user: { name: 'Alice', id: 1 } },
  status: 200,
});
```

```go
// Bad: interface mock that skips validation the real implementation performs
type mockDB struct{}

func (m *mockDB) SaveUser(u User) error {
    return nil  // Always succeeds -- real DB enforces unique email constraint
}

// Good: mock that reflects real constraints
type mockDB struct {
    users map[string]User
}

func (m *mockDB) SaveUser(u User) error {
    if _, exists := m.users[u.Email]; exists {
        return ErrDuplicateEmail  // Matches real DB behavior
    }
    m.users[u.Email] = u
    return nil
}
```

### Over-Mocking Detection

```java
// Bad: mocking everything -- test verifies mock wiring, not behavior
@Test
void processOrder() {
    when(mockValidator.validate(any())).thenReturn(true);
    when(mockPricer.calculate(any())).thenReturn(new BigDecimal("100.00"));
    when(mockInventory.reserve(any())).thenReturn(true);
    when(mockPayment.charge(any())).thenReturn(new PaymentResult("success"));
    when(mockNotifier.send(any())).thenReturn(true);

    orderService.processOrder(testOrder);

    verify(mockValidator).validate(testOrder);
    verify(mockPricer).calculate(testOrder);
    verify(mockInventory).reserve(testOrder);
    verify(mockPayment).charge(any());
    verify(mockNotifier).send(any());
    // This test only verifies that methods were called -- not that the order was processed correctly
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Mocks that return shapes inconsistent with the real dependency | HIGH |
| Mocks that always succeed, never simulating error cases | MEDIUM |
| Over-mocking: tests that verify mock call sequences instead of outcomes | MEDIUM |
| Mocks of types the test owns (suggests poor design, not a mock issue) | LOW |
| Realistic mocks with error path simulation | INFO (positive) |

---

## 7. Test Data Quality

Test data should be realistic, minimal, clearly named, and easy to maintain. Hardcoded magic values, excessive data, and opaque fixtures make tests difficult to understand and modify.

### Detection Patterns -- Poor Test Data

```python
# Bad: magic values with no context
def test_calculate_shipping():
    result = calculate_shipping(42, "90210", "10001", 2.5, True)
    assert result == 12.99
    # What do 42, "90210", "10001", 2.5, True mean?

# Good: named values that explain intent
def test_calculate_shipping_standard_domestic():
    result = calculate_shipping(
        item_count=3,
        origin_zip="90210",
        destination_zip="10001",
        weight_kg=2.5,
        is_expedited=False,
    )
    assert result == Decimal("8.99")
```

```javascript
// Bad: large opaque fixture
const testData = {
  id: 1, name: 'John', age: 30, email: 'john@test.com',
  address: { street: '123 Main', city: 'Springfield', state: 'IL', zip: '62704' },
  preferences: { theme: 'dark', notifications: true, language: 'en' },
  subscription: { plan: 'premium', startDate: '2024-01-01', autoRenew: true },
  // ... 20 more fields
};

test('getUserDisplayName', () => {
  expect(getUserDisplayName(testData)).toBe('John');
  // Only the 'name' field matters -- the rest is noise
});

// Good: only include relevant fields
test('getUserDisplayName returns the name field', () => {
  const user = { name: 'John' };
  expect(getUserDisplayName(user)).toBe('John');
});
```

```go
// Bad: unclear what makes this test data special
func TestValidate(t *testing.T) {
    input := Input{
        Name:    "asdf",
        Email:   "qwer@zxcv.com",
        Age:     99,
        Country: "XX",
    }
    err := Validate(input)
    assert.Error(t, err)
    // What was invalid? "asdf"? "XX"? 99?
}

// Good: make the interesting value obvious
func TestValidate_InvalidCountryCode(t *testing.T) {
    input := validInput()  // helper returns a known-good input
    input.Country = "XX"   // override the one field we are testing
    err := Validate(input)
    assert.ErrorContains(t, err, "invalid country code")
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Test data with magic numbers/strings that obscure test intent | MEDIUM |
| Large fixture objects where only 1-2 fields are relevant | LOW |
| Test data that is clearly named and minimal | INFO (positive) |

---

## 8. Test Fixture Management

Fixtures provide reusable test setup. Well-managed fixtures reduce duplication without hiding important context. Poorly managed fixtures become an opaque layer that makes tests harder to understand than the code under test.

### Detection Patterns -- Fixture Problems

**Over-shared fixtures**:

```python
# Bad: one giant fixture used by all tests in the file
@pytest.fixture
def setup_everything():
    db = create_test_db()
    user = create_user(db, "admin")
    product = create_product(db, "widget", 10.00)
    order = create_order(db, user, [product])
    payment = create_payment(db, order)
    return db, user, product, order, payment

def test_user_name(setup_everything):
    _, user, _, _, _ = setup_everything  # Only needs user
    assert user.name == "admin"

def test_payment_amount(setup_everything):
    _, _, _, _, payment = setup_everything  # Only needs payment
    assert payment.amount == 10.00

# Good: focused fixtures
@pytest.fixture
def user():
    return create_user("admin")

@pytest.fixture
def order(user):
    product = create_product("widget", 10.00)
    return create_order(user, [product])

def test_user_name(user):
    assert user.name == "admin"

def test_order_total(order):
    assert order.total == 10.00
```

**Fixture files that are too far from tests**:

```
# Bad: fixture defined in conftest.py four directories up
tests/
  conftest.py           # <- defines make_user
  unit/
    services/
      users/
        conftest.py
        test_user_service.py  # uses make_user but you have to hunt for it

# Good: fixture near the test that uses it, or clearly named
tests/
  conftest.py               # shared DB setup only
  unit/
    services/
      users/
        conftest.py         # <- defines make_user, close to test
        test_user_service.py
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| God fixture that sets up everything for all tests | MEDIUM |
| Fixtures defined far from their usage with no documentation | LOW |
| Focused, composable fixtures near their tests | INFO (positive) |

---

## 9. Test Helper Patterns

Test helpers extract common test logic into reusable functions. They should make tests more readable, not more opaque.

### Good Helper Patterns

```python
# Helper that creates valid defaults with overrides
def make_user(**overrides):
    defaults = {
        "name": "Test User",
        "email": "test@example.com",
        "role": "member",
    }
    defaults.update(overrides)
    return User(**defaults)

# Usage: clear and concise
def test_admin_can_delete_users():
    admin = make_user(role="admin")
    target = make_user(name="Target")
    assert admin.can_delete(target)
```

```javascript
// Helper for API test assertions
function expectSuccessResponse(response, expectedBody) {
  expect(response.status).toBe(200);
  expect(response.headers['content-type']).toMatch(/json/);
  expect(response.body).toMatchObject(expectedBody);
}

// Usage
test('GET /users/:id returns user', async () => {
  const response = await request(app).get('/api/users/1');
  expectSuccessResponse(response, { name: 'Alice' });
});
```

```go
// Helper that handles common test assertions
func requireUser(t *testing.T, db *DB, email string) User {
    t.Helper()  // Important: marks this as a helper for correct line reporting
    user, err := db.GetUserByEmail(email)
    require.NoError(t, err, "expected user with email %s to exist", email)
    return user
}
```

```java
// Builder pattern for test data
public class TestUserBuilder {
    private String name = "Test User";
    private String email = "test@example.com";
    private String role = "member";

    public TestUserBuilder withName(String name) { this.name = name; return this; }
    public TestUserBuilder withRole(String role) { this.role = role; return this; }
    public User build() { return new User(name, email, role); }
}

// Usage
@Test
void adminCanDeleteOtherUsers() {
    User admin = new TestUserBuilder().withRole("admin").build();
    User target = new TestUserBuilder().withName("Target").build();
    assertTrue(admin.canDelete(target));
}
```

### Anti-Pattern: Helper That Hides Assertions

```python
# Bad: assertions hidden inside helper -- test appears to have no assertions
def verify_user_creation(user_data):
    user = service.create_user(**user_data)
    assert user is not None
    assert user.id > 0
    assert user.name == user_data["name"]
    return user

def test_create_user():
    verify_user_creation({"name": "Alice", "email": "alice@test.com"})
    # No visible assertions in the test body
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Test helpers that hide assertions (no visible assert in test body) | MEDIUM |
| Helpers without `t.Helper()` (Go) causing wrong line numbers in failures | LOW |
| Well-structured helpers that improve readability | INFO (positive) |

---

## 10. DRY Without Over-Abstraction

Tests should avoid unnecessary duplication but not at the cost of readability. The DRY principle applies less aggressively to test code than to production code because each test should be understandable in isolation.

### The Readability Trade-Off

```python
# Over-abstracted: test is unreadable without understanding the helper
def test_standard_order():
    run_order_test("standard", 100, 0.08, 108.00)

def test_premium_order():
    run_order_test("premium", 100, 0.08, 97.20)  # What makes it 97.20?

# Better: some duplication is fine if it makes the test self-explanatory
def test_standard_order_includes_tax():
    order = create_order(item_price=100.00, customer_type="standard")
    assert order.total == 108.00  # 100.00 + 8% tax

def test_premium_order_applies_discount_then_tax():
    order = create_order(item_price=100.00, customer_type="premium")
    # 100.00 - 10% discount = 90.00 + 8% tax = 97.20
    assert order.total == 97.20
```

```javascript
// Over-abstracted: parameterized test obscures the logic
const cases = [
  ['standard', 100, 108],
  ['premium', 100, 97.2],
  ['vip', 100, 90],
];
test.each(cases)('order type %s: $%d -> $%d', (type, price, expected) => {
  expect(createOrder(price, type).total).toBe(expected);
  // Why does premium become 97.2? Reader has to reverse-engineer it
});

// Better: table-driven but with explanatory names
test.each([
  { type: 'standard', price: 100, expected: 108, reason: 'base + 8% tax' },
  { type: 'premium', price: 100, expected: 97.2, reason: '10% discount then 8% tax' },
  { type: 'vip', price: 100, expected: 90, reason: '10% discount, tax exempt' },
])('$type order: $reason', ({ type, price, expected }) => {
  expect(createOrder(price, type).total).toBe(expected);
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Parameterized tests where the expected values are unexplained magic numbers | MEDIUM |
| Test code duplicated more than 3 times with no extraction | LOW |
| Tests that are self-explanatory with appropriate helper use | INFO (positive) |

---

## 11. Page Objects for UI Tests

Page objects encapsulate page structure and interactions, isolating tests from DOM/UI implementation details. When the UI changes, only the page object needs updating.

### Detection Patterns -- Missing Page Objects

```python
# Bad: selectors and interactions scattered in tests
def test_login(browser):
    browser.find_element(By.ID, "username").send_keys("admin")
    browser.find_element(By.ID, "password").send_keys("secret")
    browser.find_element(By.CSS_SELECTOR, "button.login-btn").click()
    assert browser.find_element(By.CSS_SELECTOR, ".dashboard-title").text == "Dashboard"

def test_login_invalid_password(browser):
    browser.find_element(By.ID, "username").send_keys("admin")
    browser.find_element(By.ID, "password").send_keys("wrong")
    browser.find_element(By.CSS_SELECTOR, "button.login-btn").click()
    assert browser.find_element(By.CSS_SELECTOR, ".error-message").is_displayed()

# Good: page object encapsulates selectors
class LoginPage:
    def __init__(self, browser):
        self.browser = browser

    def login(self, username, password):
        self.browser.find_element(By.ID, "username").send_keys(username)
        self.browser.find_element(By.ID, "password").send_keys(password)
        self.browser.find_element(By.CSS_SELECTOR, "button.login-btn").click()

    @property
    def error_message(self):
        return self.browser.find_element(By.CSS_SELECTOR, ".error-message")

    @property
    def is_dashboard_visible(self):
        return self.browser.find_element(By.CSS_SELECTOR, ".dashboard-title").is_displayed()

def test_login(login_page):
    login_page.login("admin", "secret")
    assert login_page.is_dashboard_visible

def test_login_invalid_password(login_page):
    login_page.login("admin", "wrong")
    assert login_page.error_message.is_displayed()
```

```javascript
// Good: Playwright page object
class LoginPage {
  constructor(page) {
    this.page = page;
    this.usernameInput = page.locator('[data-testid="username"]');
    this.passwordInput = page.locator('[data-testid="password"]');
    this.submitButton = page.locator('[data-testid="submit"]');
    this.errorMessage = page.locator('[data-testid="error"]');
    this.dashboard = page.locator('[data-testid="dashboard"]');
  }

  async login(username, password) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| UI tests with selectors duplicated across >5 test functions | MEDIUM |
| No page objects in a project with >10 UI tests | MEDIUM |
| Selectors using auto-generated IDs or deep CSS paths | MEDIUM |
| Well-implemented page objects with stable test IDs | INFO (positive) |

---

## 12. Factory Patterns for Test Data

Factories create test objects with sensible defaults, allowing tests to specify only the fields relevant to the scenario. They are the preferred pattern for managing test data in non-trivial test suites.

### Language-Specific Implementations

**Python (factory_boy)**:

```python
import factory
from myapp.models import User, Order

class UserFactory(factory.Factory):
    class Meta:
        model = User

    name = factory.Faker('name')
    email = factory.LazyAttribute(lambda obj: f"{obj.name.lower().replace(' ', '.')}@test.com")
    role = "member"
    is_active = True

class OrderFactory(factory.Factory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)
    total = factory.Faker('pydecimal', left_digits=3, right_digits=2, positive=True)
    status = "pending"

# Usage: override only what matters
def test_admin_sees_all_orders():
    admin = UserFactory(role="admin")
    orders = [OrderFactory() for _ in range(5)]
    visible = get_visible_orders(admin)
    assert len(visible) == 5
```

**JavaScript/TypeScript (fishery or manual)**:

```typescript
// factories.ts using fishery
import { Factory } from 'fishery';

interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'member';
}

const userFactory = Factory.define<User>(({ sequence }) => ({
  id: sequence,
  name: `User ${sequence}`,
  email: `user${sequence}@test.com`,
  role: 'member',
}));

// Usage
const admin = userFactory.build({ role: 'admin' });
const users = userFactory.buildList(5);
```

**Go (manual helper)**:

```go
// testutil/factories.go
func NewTestUser(overrides ...func(*User)) User {
    u := User{
        Name:  "Test User",
        Email: "test@example.com",
        Role:  "member",
        Active: true,
    }
    for _, override := range overrides {
        override(&u)
    }
    return u
}

func WithRole(role string) func(*User) {
    return func(u *User) { u.Role = role }
}

func WithEmail(email string) func(*User) {
    return func(u *User) { u.Email = email }
}

// Usage
func TestAdminAccess(t *testing.T) {
    admin := NewTestUser(WithRole("admin"))
    member := NewTestUser(WithRole("member"))
    assert.True(t, CanDeleteUser(admin, member))
}
```

**Java (manual builder or Instancio)**:

```java
// TestDataFactory.java
public class TestData {
    public static User.Builder aUser() {
        return User.builder()
            .name("Test User")
            .email("test@example.com")
            .role("member")
            .active(true);
    }

    public static Order.Builder anOrder() {
        return Order.builder()
            .user(aUser().build())
            .total(new BigDecimal("100.00"))
            .status(OrderStatus.PENDING);
    }
}

// Usage
@Test
void adminCanDeleteUsers() {
    User admin = TestData.aUser().role("admin").build();
    User target = TestData.aUser().build();
    assertTrue(userService.canDelete(admin, target));
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Test data constructed inline with many hardcoded fields across multiple tests | MEDIUM |
| No factory pattern in a project with >50 tests creating similar objects | LOW |
| Well-implemented factories with sensible defaults | INFO (positive) |

---

## 13. Flaky Test Detection Signals

Flaky tests are tests that sometimes pass and sometimes fail without any code change. They erode trust in the test suite, cause developers to ignore failures, and waste CI time with retries.

### Signal 1: Sleep / Hardcoded Delays

```python
# Flaky: timing-dependent
def test_async_processing():
    submit_job(job_data)
    time.sleep(5)  # Hope it finishes in 5 seconds
    result = get_job_result(job_id)
    assert result.status == "complete"

# Better: poll with timeout
def test_async_processing():
    submit_job(job_data)
    result = poll_until(
        lambda: get_job_result(job_id),
        predicate=lambda r: r.status == "complete",
        timeout=30,
        interval=0.5,
    )
    assert result.status == "complete"
```

```javascript
// Flaky: arbitrary timeout
test('notification appears', async () => {
  triggerNotification();
  await new Promise(resolve => setTimeout(resolve, 2000));
  expect(screen.getByText('Success')).toBeVisible();
});

// Better: wait for condition
test('notification appears', async () => {
  triggerNotification();
  await waitFor(() => {
    expect(screen.getByText('Success')).toBeVisible();
  });
});
```

### Signal 2: Time-Dependent Tests

```python
# Flaky: fails near midnight or during DST transitions
def test_is_business_hours():
    now = datetime.now()
    assert is_business_hours(now) == True  # Fails at night, on weekends

# Better: inject the time
def test_is_business_hours_tuesday_10am():
    tuesday_10am = datetime(2024, 3, 5, 10, 0, 0)  # Known Tuesday
    assert is_business_hours(tuesday_10am) == True
```

```go
// Flaky: uses real clock
func TestCacheExpiry(t *testing.T) {
    cache := NewCache(ttl: time.Second)
    cache.Set("key", "value")
    time.Sleep(1100 * time.Millisecond)  // Flaky: sleep may not be precise
    _, ok := cache.Get("key")
    assert.False(t, ok)
}

// Better: injectable clock
func TestCacheExpiry(t *testing.T) {
    clock := NewFakeClock(time.Now())
    cache := NewCache(ttl: time.Second, clock: clock)
    cache.Set("key", "value")
    clock.Advance(2 * time.Second)
    _, ok := cache.Get("key")
    assert.False(t, ok)
}
```

### Signal 3: Network-Dependent Tests

```java
// Flaky: depends on external service availability
@Test
void fetchExchangeRate() {
    ExchangeRate rate = client.getRate("USD", "EUR");
    assertEquals(0.85, rate.getValue(), 0.1);  // Also flaky: rate changes daily
}

// Better: use WireMock or similar
@Test
void fetchExchangeRate() {
    wireMock.stubFor(get("/rates/USD/EUR")
        .willReturn(jsonResponse("""{"rate": 0.85}""")));

    ExchangeRate rate = client.getRate("USD", "EUR");
    assertEquals(0.85, rate.getValue(), 0.001);
}
```

### Signal 4: Order-Dependent Tests

See Section 4 (Test Independence and Isolation) for detailed patterns.

### Signal 5: Random Without Seed

```python
# Flaky: random input is different every run
def test_sort_handles_random_input():
    data = [random.randint(0, 1000) for _ in range(100)]
    result = custom_sort(data)
    assert result == sorted(data)
    # When this fails, you cannot reproduce it

# Better: use a seed and log it
def test_sort_handles_random_input():
    seed = int(time.time())
    random.seed(seed)
    data = [random.randint(0, 1000) for _ in range(100)]
    result = custom_sort(data)
    assert result == sorted(data), f"Failed with seed={seed}"
```

```javascript
// Better in JS: use a seeded PRNG
const seedrandom = require('seedrandom');

test('shuffle produces valid permutation', () => {
  const seed = Date.now().toString();
  const rng = seedrandom(seed);
  const input = [1, 2, 3, 4, 5];
  const result = shuffle(input, rng);
  expect(result.sort()).toEqual(input.sort());
  // If test fails, report: `Seed: ${seed}`
});
```

### Signal Summary

| Signal | Detection Pattern | Severity |
|--------|-------------------|----------|
| `time.sleep()`, `setTimeout()`, `Thread.sleep()` in tests | Hardcoded wait | MEDIUM |
| `datetime.now()`, `Date.now()`, `time.Now()` in assertions | Wall-clock dependency | MEDIUM |
| HTTP/gRPC calls to external services without mocks | Network dependency | HIGH |
| Module-level mutable state, `global`, `static` in test files | Order dependency | HIGH |
| `random.random()`, `Math.random()` without seed | Non-reproducible | MEDIUM |
| Filesystem operations without temp directory isolation | Environment dependency | MEDIUM |
| Tests that only fail in CI but pass locally | Environment-specific | HIGH |
| Tests with `@retry`, `flaky`, or `skip` annotations | Known flaky | HIGH (if left unresolved) |

---

## Quick-Reference Summary

| Check | Key Signal | Severity |
|-------|-----------|----------|
| Tests with interleaved act-assert cycles | >3 cycles per test | MEDIUM |
| Tests with nondescript names (test1, testIt) | No scenario/expectation | MEDIUM |
| Tests that fail when run in random order | Order dependency | HIGH |
| Tests with no assertions | Coverage-only tests | HIGH |
| Only truthiness/nullability assertions | Weak verification | MEDIUM |
| Mocks that never simulate errors | Happy-path-only mocks | MEDIUM |
| Mocks returning wrong shapes | Contract mismatch | HIGH |
| Test data with unexplained magic values | Opaque intent | MEDIUM |
| God fixture used by all tests | Over-shared setup | MEDIUM |
| Parameterized tests with magic expected values | DRY over readability | MEDIUM |
| UI tests with duplicated selectors, no page objects | Brittle UI tests | MEDIUM |
| No factory pattern with >50 similar object constructions | Manual object building | LOW |
| `time.sleep` / `Thread.sleep` in tests | Flaky signal | MEDIUM |
| `datetime.now()` in assertions | Time dependency | MEDIUM |
| External HTTP calls without mocks | Network dependency | HIGH |
| `random` without seed | Non-reproducible | MEDIUM |
| Tests marked `@retry` or `@flaky` without fix plan | Known-flaky debt | HIGH |
