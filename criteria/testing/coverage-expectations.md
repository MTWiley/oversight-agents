# Coverage Expectations Reference

This reference provides detailed criteria for evaluating test coverage across codebases. It complements the inline criteria in `review-testing.md` with expanded definitions, threshold guidance, tool configuration, and language-specific examples.

The `testing-reviewer` agent uses this file as a lookup when evaluating coverage. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Coverage Types

Understanding the distinction between coverage types is essential for meaningful assessment. Line coverage alone creates a false sense of security; branch and path coverage expose gaps that line coverage misses entirely.

### Line Coverage

**Definition**: The percentage of executable source lines exercised by at least one test. This is the most common metric and the easiest to game.

**Limitations**: A line can be "covered" without its behavior being verified. A test that calls a function but never asserts on its output will report the function's lines as covered.

**Detection pattern -- false coverage**:

```python
# This test "covers" the function but verifies nothing
def test_calculate_discount():
    calculate_discount(100, 0.2)  # No assertion -- line coverage says 100%
```

```javascript
// Covered but unverified
test('processOrder', () => {
  processOrder({ items: [{ price: 10, qty: 1 }] });
  // No expect() -- Jest reports lines as covered
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Line coverage reported without branch coverage, presented as sufficient | MEDIUM |
| Tests that cover lines but have no assertions (coverage inflation) | HIGH |
| Line coverage below project minimum threshold | MEDIUM |

---

### Branch Coverage

**Definition**: The percentage of conditional branches (both true and false paths of `if`, `else`, ternary, `switch`/`case`, short-circuit `&&`/`||`) exercised by tests. Branch coverage is strictly more informative than line coverage.

**Why it matters**: A single-line conditional like `return x > 0 ? x : -x` has two branches. Line coverage reports 100% if either branch is taken. Branch coverage requires both.

**Detection pattern -- missing branch coverage**:

```python
def categorize_age(age):
    if age < 0:
        raise ValueError("Age cannot be negative")
    elif age < 18:
        return "minor"
    elif age < 65:
        return "adult"
    else:
        return "senior"

# This test covers only 2 of 4 branches
def test_categorize_age():
    assert categorize_age(25) == "adult"
    assert categorize_age(70) == "senior"
    # Missing: age < 0 (error branch), age < 18 (minor branch)
```

```go
func Categorize(age int) (string, error) {
    if age < 0 {
        return "", fmt.Errorf("age cannot be negative")
    }
    if age < 18 {
        return "minor", nil
    }
    if age < 65 {
        return "adult", nil
    }
    return "senior", nil
}

// Missing error path test -- only happy paths tested
func TestCategorize(t *testing.T) {
    got, _ := Categorize(30)
    if got != "adult" {
        t.Errorf("got %s, want adult", got)
    }
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| No branch coverage measurement configured when the project has conditional logic | MEDIUM |
| Error-handling branches consistently untested | HIGH |
| Branch coverage below 60% on code with significant conditional logic | MEDIUM |

---

### Path Coverage

**Definition**: The percentage of all possible execution paths through a function that are exercised. Path coverage is the most thorough metric but is impractical to achieve 100% in functions with many conditionals (paths grow exponentially).

**When to require**: Path coverage is most valuable for small, critical functions where the interaction between conditions matters (e.g., validation logic, state machines, permission checks).

**Example -- path explosion**:

```java
// This function has 2^3 = 8 possible paths
public String evaluate(boolean a, boolean b, boolean c) {
    String result = "";
    if (a) result += "A";
    if (b) result += "B";
    if (c) result += "C";
    return result;
}

// Full path coverage requires 8 tests:
// (F,F,F), (T,F,F), (F,T,F), (F,F,T), (T,T,F), (T,F,T), (F,T,T), (T,T,T)
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Critical validation/auth functions without path coverage consideration | HIGH |
| Combinatorial explosion in a single function suggesting need for decomposition | MEDIUM |
| Path coverage reported for non-critical utility code (over-investment) | INFO |

---

## 2. Coverage Thresholds by Project Type

Coverage expectations vary significantly by project type, maturity, and risk profile. Applying the same threshold to a throwaway script and a payment processing service is counterproductive.

### Threshold Matrix

| Project Type | Line Coverage | Branch Coverage | Critical Path Coverage | Rationale |
|-------------|--------------|----------------|----------------------|-----------|
| Payment / Financial | >= 90% | >= 85% | >= 95% | Regulatory requirements, financial impact of bugs |
| Authentication / Authorization | >= 90% | >= 85% | >= 95% | Security impact, credential handling |
| Healthcare / Medical | >= 90% | >= 85% | >= 95% | Regulatory (HIPAA, FDA), patient safety |
| API / Web Service | >= 80% | >= 70% | >= 90% | Customer-facing, SLA obligations |
| Library / Framework | >= 85% | >= 75% | >= 90% | Consumed by others, contract stability |
| CLI Tool | >= 75% | >= 65% | >= 85% | User-facing, input parsing complexity |
| Internal Tool | >= 70% | >= 60% | >= 80% | Lower blast radius, faster iteration |
| Prototype / PoC | >= 50% | N/A | >= 70% (if critical paths exist) | Speed over thoroughness, will be rewritten |
| Infrastructure / IaC | >= 70% | >= 60% | >= 85% | Production impact, drift detection |
| Data Pipeline | >= 75% | >= 65% | >= 85% | Data integrity, silent corruption risk |

### Severity Classification for Threshold Violations

| Gap from Threshold | Severity |
|-------------------|----------|
| > 20 percentage points below target | HIGH |
| 10-20 percentage points below target | MEDIUM |
| 5-10 percentage points below target | LOW |
| Within 5 points of target | INFO (trending note) |
| At or above target | INFO (positive callout) |

### New Code vs. Existing Code

New code in a pull request should meet or exceed the project's coverage threshold regardless of the overall project coverage level. This prevents coverage erosion.

```yaml
# Example: diff-coverage configuration (diff-cover tool)
# Require 80% coverage on changed lines, even if the project average is 60%
diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| New code in a PR has 0% coverage | HIGH |
| New code coverage is below the project threshold | MEDIUM |
| New code coverage meets the threshold but overall project coverage dropped | LOW |
| Coverage improved with this change | INFO (positive) |

---

## 3. Critical Path Coverage Requirements

Certain code paths carry disproportionate risk. These paths must meet higher coverage standards regardless of the project's general threshold.

### Authentication and Authorization

**Required coverage**: >= 95% branch coverage.

Every authentication decision point -- login, token validation, session management, password reset, MFA verification -- must have tests for both the success and failure paths.

**Detection pattern -- missing auth coverage**:

```python
# auth.py -- all branches must be tested
def authenticate(username, password):
    user = get_user(username)
    if user is None:                    # Branch: user not found
        raise AuthError("Invalid credentials")
    if user.is_locked:                  # Branch: account locked
        raise AuthError("Account locked")
    if not verify_password(password, user.password_hash):  # Branch: wrong password
        increment_failed_attempts(user)
        raise AuthError("Invalid credentials")
    if user.requires_mfa:               # Branch: MFA required
        return {"status": "mfa_required", "token": generate_mfa_token(user)}
    return {"status": "authenticated", "token": generate_session_token(user)}

# Required test cases (minimum):
# 1. Valid credentials -> success
# 2. Unknown user -> AuthError
# 3. Locked account -> AuthError
# 4. Wrong password -> AuthError + failed attempt increment
# 5. Valid credentials + MFA required -> mfa_required status
```

```java
// Spring Security -- verify filter chain coverage
@Test
void authenticatedUserCanAccessProtectedResource() { ... }

@Test
void unauthenticatedUserReceives401() { ... }

@Test
void userWithWrongRoleReceives403() { ... }

@Test
void expiredTokenReceives401() { ... }

@Test
void malformedTokenReceives401() { ... }
```

### Payment and Financial Operations

**Required coverage**: >= 95% branch coverage.

All money-movement code, including edge cases around currency precision, rounding, refunds, partial payments, and idempotency.

**Detection pattern -- insufficient payment coverage**:

```python
# payment.py -- critical path
def process_payment(amount, currency, payment_method):
    if amount <= 0:
        raise ValueError("Amount must be positive")
    if amount > MAX_TRANSACTION_AMOUNT:
        raise ValueError("Amount exceeds maximum")

    charge = gateway.charge(amount, currency, payment_method)

    if charge.status == "declined":
        return PaymentResult(status="declined", reason=charge.decline_reason)
    if charge.status == "requires_action":
        return PaymentResult(status="pending", action_url=charge.action_url)

    record_transaction(charge)
    return PaymentResult(status="success", transaction_id=charge.id)

# Required test cases:
# 1. Successful payment
# 2. Zero amount -> ValueError
# 3. Negative amount -> ValueError
# 4. Amount exceeds maximum -> ValueError
# 5. Declined charge -> declined result with reason
# 6. Requires action (3D Secure) -> pending result with URL
# 7. Gateway timeout/error -> appropriate error handling
# 8. Idempotency: duplicate submission returns same result
# 9. Currency precision (e.g., JPY has 0 decimal places, USD has 2)
```

### Data Mutation Operations

**Required coverage**: >= 90% branch coverage.

Any operation that creates, updates, or deletes persistent data must be tested for success, validation failure, constraint violation, and concurrent modification scenarios.

**Detection pattern -- missing mutation coverage**:

```go
// Missing test for partial failure scenario
func (s *Service) TransferFunds(from, to string, amount float64) error {
    tx, err := s.db.Begin()
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()

    if err := s.debit(tx, from, amount); err != nil {
        return fmt.Errorf("debit: %w", err)  // Must test this path
    }
    if err := s.credit(tx, to, amount); err != nil {
        return fmt.Errorf("credit: %w", err)  // Must test this path
    }

    return tx.Commit()
}

// Required tests:
// 1. Successful transfer
// 2. Begin transaction fails
// 3. Debit fails (insufficient funds, account not found)
// 4. Credit fails after successful debit (rollback verification)
// 5. Commit fails (rollback verification)
```

**Severity guide for critical path gaps**:

| Condition | Severity |
|-----------|----------|
| Authentication code with untested branches | CRITICAL |
| Payment processing with untested error paths | CRITICAL |
| Data mutation with no rollback/failure testing | HIGH |
| Critical path below 90% branch coverage | HIGH |
| Critical path below 95% but above 90% | MEDIUM |

---

## 4. Coverage Measurement Tools and Configuration

### Tool Reference by Language

| Language | Tool | Configuration File | Command |
|----------|------|--------------------|---------|
| Python | coverage.py / pytest-cov | `.coveragerc` or `pyproject.toml` | `pytest --cov=src --cov-branch --cov-report=xml` |
| JavaScript/TypeScript | Istanbul / nyc / c8 / Jest | `.nycrc` or `jest.config.js` | `jest --coverage --coverageReporters=lcov` |
| Go | go test -cover | N/A (built-in) | `go test -coverprofile=coverage.out -covermode=atomic ./...` |
| Java | JaCoCo | `build.gradle` or `pom.xml` | `./gradlew jacocoTestReport` |
| Ruby | SimpleCov | `.simplecov` or `spec_helper.rb` | `COVERAGE=true bundle exec rspec` |
| Rust | cargo-tarpaulin / llvm-cov | `tarpaulin.toml` | `cargo tarpaulin --out xml` |
| C# | coverlet / dotCover | `coverlet.runsettings` | `dotnet test --collect:"XPlat Code Coverage"` |

### Configuration Best Practices

**Python -- pytest-cov with branch coverage**:

```toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
branch = true
omit = [
    "*/tests/*",
    "*/migrations/*",
    "*/__pycache__/*",
    "*/conftest.py",
]

[tool.coverage.report]
fail_under = 80
show_missing = true
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "if __name__ == .__main__.",
    "@overload",
    "raise NotImplementedError",
    "\\.\\.\\.",
]
```

**JavaScript/TypeScript -- Jest coverage configuration**:

```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'json-summary'],
  collectCoverageFrom: [
    'src/**/*.{js,ts}',
    '!src/**/*.d.ts',
    '!src/**/*.test.{js,ts}',
    '!src/**/index.{js,ts}',  // barrel files
    '!src/**/__mocks__/**',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // Higher thresholds for critical paths
    './src/auth/': {
      branches: 90,
      functions: 95,
      lines: 95,
    },
    './src/payment/': {
      branches: 90,
      functions: 95,
      lines: 95,
    },
  },
};
```

**Go -- coverage with atomic mode for concurrent code**:

```bash
# Atomic mode is required for accurate coverage of concurrent code
go test -coverprofile=coverage.out -covermode=atomic ./...

# Generate HTML report for local review
go tool cover -html=coverage.out -o coverage.html

# Check coverage threshold (use a script or tool like go-coverage-threshold)
go tool cover -func=coverage.out | grep total | awk '{print $3}'
```

```go
// go.mod project -- ensure test build tags include integration tests
//go:build integration

package mypackage_test
// Integration tests run separately with: go test -tags=integration ./...
```

**Java -- JaCoCo with Gradle**:

```groovy
// build.gradle
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/generated/**',
                '**/config/**',
                '**/model/*DTO.*',
            ])
        }))
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = 'BRANCH'
                minimum = 0.70
            }
            limit {
                counter = 'LINE'
                minimum = 0.80
            }
        }
        // Higher threshold for critical packages
        rule {
            element = 'PACKAGE'
            includes = ['com.example.auth.*', 'com.example.payment.*']
            limit {
                counter = 'BRANCH'
                minimum = 0.90
            }
        }
    }
}

check.dependsOn jacocoTestCoverageVerification
```

**Severity guide for tool configuration issues**:

| Condition | Severity |
|-----------|----------|
| No coverage tool configured in a production project | HIGH |
| Coverage tool present but branch coverage not enabled | MEDIUM |
| Coverage tool configured but thresholds not enforced | MEDIUM |
| Coverage tool configured with thresholds but not integrated into CI | MEDIUM |
| Well-configured coverage with branch analysis and CI enforcement | INFO (positive) |

---

## 5. Coverage Exclusion Justification

Not all code warrants coverage. Valid exclusions exist, but every exclusion should be justified and auditable. Unjustified exclusions erode coverage quality silently.

### Valid Exclusion Categories

| Category | Example | Justification |
|----------|---------|---------------|
| Type-checking blocks | `if TYPE_CHECKING:` (Python) | Only used by static analyzers, never executes at runtime |
| Defensive unreachable code | `raise NotImplementedError` in abstract methods | Exists as a safety net, not a reachable path |
| Auto-generated code | ORM migrations, protobuf stubs, OpenAPI clients | Generated code should be validated by the generator's tests |
| Framework boilerplate | `if __name__ == "__main__":` guards | Entry points tested via integration tests, not unit tests |
| Debug/development utilities | Development-only logging, REPL helpers | Not shipped to production |
| Platform-specific branches | OS-specific code on a platform not in CI | Cannot execute in the CI environment (but should be tested in a platform-specific CI job if possible) |

### Invalid Exclusion Patterns

**Detection pattern -- suspicious exclusions**:

```python
# Suspicious: excluding error handling from coverage
def process_data(data):
    try:
        result = transform(data)
    except ValueError:  # pragma: no cover   <-- This SHOULD be tested
        logger.error("Transform failed")
        raise
    return result
```

```javascript
// Suspicious: excluding entire files from coverage
// jest.config.js
collectCoverageFrom: [
  'src/**/*.{js,ts}',
  '!src/services/**',  // Excluding all services -- WHY?
  '!src/controllers/**',  // Excluding all controllers -- WHY?
]
```

```go
// Suspicious: skipping test for complex function
func ComplexBusinessLogic(input Input) (Output, error) {
    // 50 lines of conditional logic
    // No test file exists for this package
}
```

**Severity guide for exclusions**:

| Condition | Severity |
|-----------|----------|
| Error-handling code excluded from coverage | HIGH |
| Business logic files excluded from coverage without justification | HIGH |
| Large exclusion lists (>20% of source files excluded) | MEDIUM |
| Valid exclusions without inline justification comments | LOW |
| Well-documented exclusions with clear rationale | INFO (positive) |

### Exclusion Audit Pattern

Every exclusion pragma should have a justification comment:

```python
# Good: justified exclusion
if TYPE_CHECKING:  # pragma: no cover -- type-checking-only imports
    from heavy_module import HeavyType

# Good: justified exclusion
def __repr__(self):  # pragma: no cover -- debug representation
    return f"Order(id={self.id}, total={self.total})"

# Bad: unjustified exclusion
def handle_timeout(self):  # pragma: no cover
    self.retry()
```

---

## 6. Integration vs. Unit Coverage Balance

A healthy test suite balances fast, isolated unit tests with slower integration tests that verify component interactions. Over-indexing on either creates blind spots.

### Coverage Distribution Guidelines

| Test Type | Ideal Share of Tests | Coverage Focus | Execution Speed |
|-----------|---------------------|----------------|-----------------|
| Unit tests | 60-70% | Business logic, algorithms, validation, pure functions | Milliseconds per test |
| Integration tests | 20-30% | Database queries, API contracts, service interactions, message handling | Seconds per test |
| End-to-end tests | 5-15% | Critical user workflows, smoke tests, deployment validation | Seconds to minutes per test |

### Detection Patterns -- Imbalanced Coverage

**Over-reliance on unit tests (missing integration coverage)**:

```python
# Unit test mocks the database entirely -- covers lines but misses real behavior
def test_create_user(mock_db):
    mock_db.insert.return_value = {"id": 1, "name": "Alice"}
    result = create_user("Alice", "alice@example.com")
    assert result["name"] == "Alice"
    mock_db.insert.assert_called_once()
    # But: Does the actual SQL work? Are constraints enforced?
    # Missing: integration test with a real (test) database
```

```javascript
// All tests mock HTTP calls -- no actual API integration tests
jest.mock('axios');
test('fetchUser returns user data', async () => {
  axios.get.mockResolvedValue({ data: { id: 1, name: 'Alice' } });
  const user = await fetchUser(1);
  expect(user.name).toBe('Alice');
  // But: Does the real API return this shape? Is auth handled correctly?
});
```

**Over-reliance on integration tests (slow, brittle suite)**:

```java
// Every test spins up the full Spring context
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    @Test
    void createUser() {
        // This starts the entire application for a simple validation test
        // Pure validation logic should be unit-tested without Spring
    }
}
```

```go
// Every test connects to a real database
func TestValidateEmail(t *testing.T) {
    db := setupTestDB(t)  // 500ms setup for a pure function test
    defer db.Close()
    // validateEmail is a pure function -- does not need a database
    if !validateEmail("test@example.com") {
        t.Fatal("expected valid email")
    }
}
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Database-interacting code with only mocked unit tests and no integration tests | HIGH |
| API clients with no contract or integration tests | MEDIUM |
| Pure logic tested only through slow integration tests | MEDIUM |
| No integration tests in a project with external dependencies | HIGH |
| Balanced test pyramid with appropriate coverage at each level | INFO (positive) |

---

## 7. End-to-End Coverage for Critical User Workflows

End-to-end (E2E) tests validate that the system works correctly from the user's perspective. Not every feature needs E2E coverage, but critical workflows must have it.

### Workflows That Require E2E Coverage

| Workflow Category | Examples | Severity if Missing |
|-------------------|----------|-------------------|
| Authentication flow | Login, logout, registration, password reset, MFA | CRITICAL |
| Payment flow | Checkout, payment processing, refund | CRITICAL |
| Data export/import | Report generation, bulk import, data export | HIGH |
| Onboarding / setup | Account creation, initial configuration | HIGH |
| Core business transaction | The primary value-delivering action of the application | HIGH |
| Admin operations | User management, configuration changes, permission grants | MEDIUM |
| Search and retrieval | Primary search workflows, filtering, pagination | MEDIUM |

### E2E Test Quality Checks

**Detection pattern -- brittle E2E tests**:

```javascript
// Bad: Selectors coupled to implementation details
test('user can log in', async () => {
  await page.click('div.MuiButton-root:nth-child(3)');  // Fragile CSS selector
  await page.fill('#input-27', 'admin');  // Auto-generated ID
  await page.waitForTimeout(3000);  // Arbitrary sleep
});

// Good: Stable selectors and meaningful waits
test('user can log in', async () => {
  await page.click('[data-testid="login-button"]');
  await page.fill('[data-testid="username-input"]', 'admin');
  await page.fill('[data-testid="password-input"]', 'password');
  await page.click('[data-testid="submit-button"]');
  await expect(page.locator('[data-testid="dashboard"]')).toBeVisible();
});
```

```python
# Bad: Fragile XPath and hardcoded waits
def test_checkout(browser):
    browser.find_element(By.XPATH, '/html/body/div[3]/div/div[2]/button').click()
    time.sleep(5)

# Good: Page object pattern with explicit waits
def test_checkout(checkout_page):
    checkout_page.enter_payment_details(VALID_CARD)
    checkout_page.submit_order()
    assert checkout_page.confirmation_message.is_displayed()
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| No E2E tests for authentication or payment workflows | CRITICAL |
| E2E tests exist but are disabled due to flakiness | HIGH |
| Core business workflow has no E2E coverage | HIGH |
| E2E tests use fragile selectors (auto-generated IDs, deep CSS paths) | MEDIUM |
| E2E tests use hardcoded sleeps instead of explicit waits | MEDIUM |
| Well-structured E2E suite with page objects and stable selectors | INFO (positive) |

---

## 8. Coverage Reporting in CI

Coverage data is only useful if it is visible, enforced, and tracked over time. Coverage reports buried in build logs are effectively nonexistent.

### CI Integration Requirements

| Requirement | Description | Severity if Missing |
|-------------|-------------|-------------------|
| Coverage runs on every PR | Coverage report generated for every pull request | HIGH |
| Coverage threshold enforced | Build fails if coverage drops below threshold | MEDIUM |
| Diff coverage reported | Coverage of changed lines specifically highlighted | MEDIUM |
| Coverage trend tracked | Historical coverage data stored and visualized | LOW |
| Coverage visible in PR | Coverage summary posted as PR comment or check | LOW |
| Branch coverage included | Report includes branch coverage, not just line coverage | MEDIUM |

### Configuration Examples

**GitHub Actions with coverage enforcement**:

```yaml
# .github/workflows/test.yml
name: Test
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: pytest --cov=src --cov-branch --cov-report=xml --cov-fail-under=80
      - name: Diff coverage check
        run: |
          pip install diff-cover
          diff-cover coverage.xml --compare-branch=origin/main --fail-under=80
      - name: Upload coverage to tracking service
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml
          fail_ci_if_error: true
```

**GitLab CI with coverage parsing**:

```yaml
# .gitlab-ci.yml
test:
  stage: test
  script:
    - pytest --cov=src --cov-branch --cov-report=term-missing --cov-report=xml
  coverage: '/^TOTAL\s+\d+\s+\d+\s+\d+\s+\d+\s+([\d.]+%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```

**Go coverage in CI**:

```yaml
# GitHub Actions
- name: Run tests with coverage
  run: |
    go test -coverprofile=coverage.out -covermode=atomic ./...
    COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
    echo "Total coverage: ${COVERAGE}%"
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "Coverage ${COVERAGE}% is below 80% threshold"
      exit 1
    fi
```

**Java JaCoCo in CI (Gradle)**:

```yaml
# GitHub Actions
- name: Run tests and check coverage
  run: ./gradlew test jacocoTestReport jacocoTestCoverageVerification
- name: Publish coverage report
  uses: actions/upload-artifact@v4
  with:
    name: coverage-report
    path: build/reports/jacoco/test/html/
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| No coverage measurement in CI pipeline | HIGH |
| Coverage measured but not enforced (no fail threshold) | MEDIUM |
| Coverage enforced but no diff-coverage check (allows legacy debt to mask new gaps) | MEDIUM |
| Coverage reported only as a build log line (not visible in PR review) | LOW |
| Full coverage pipeline: measurement, enforcement, diff-check, trend tracking | INFO (positive) |

---

## 9. Coverage Regression Detection

Coverage should trend upward or remain stable. Any decrease deserves investigation because it may indicate untested new code, deleted tests, or test infrastructure problems.

### Regression Detection Strategies

**Strategy 1: Diff coverage enforcement**

The most practical approach. Rather than tracking absolute coverage, require that new/changed lines meet the coverage threshold. This prevents coverage erosion without requiring teams to retroactively improve legacy code.

```bash
# Python: diff-cover
diff-cover coverage.xml --compare-branch=origin/main --fail-under=80

# JavaScript: jest with changedSince
jest --changedSince=origin/main --coverage
```

**Strategy 2: Ratchet mechanism**

Record the current coverage percentage and require that each merge does not decrease it. This ensures monotonic improvement.

```python
# .coverage-threshold (checked into repo, updated by CI)
# Current minimum: 78.3
# This value only goes up, never down
MINIMUM_COVERAGE=78.3
```

```bash
#!/bin/bash
# coverage-ratchet.sh -- run after tests
CURRENT=$(python -c "import json; d=json.load(open('coverage.json')); print(d['totals']['percent_covered'])")
MINIMUM=$(cat .coverage-threshold | grep MINIMUM_COVERAGE | cut -d= -f2)

if (( $(echo "$CURRENT < $MINIMUM" | bc -l) )); then
    echo "FAIL: Coverage dropped from ${MINIMUM}% to ${CURRENT}%"
    exit 1
fi

# Update the threshold if coverage improved
if (( $(echo "$CURRENT > $MINIMUM" | bc -l) )); then
    echo "Coverage improved from ${MINIMUM}% to ${CURRENT}% -- updating threshold"
    sed -i "s/MINIMUM_COVERAGE=.*/MINIMUM_COVERAGE=${CURRENT}/" .coverage-threshold
fi
```

**Strategy 3: Coverage trend monitoring**

Use a service (Codecov, Coveralls, SonarQube) to track coverage over time and alert on decreases.

```yaml
# codecov.yml
coverage:
  status:
    project:
      default:
        target: auto     # Use previous coverage as the target
        threshold: 1%    # Allow up to 1% decrease (to account for measurement variance)
    patch:
      default:
        target: 80%      # New code must have 80% coverage
```

### Detection Patterns -- Coverage Regression Signals

| Signal | What It Means | Severity |
|--------|---------------|----------|
| Large PR with 0% diff coverage | New code entirely untested | HIGH |
| Coverage dropped > 2% in a single PR | Significant untested code added or tests deleted | HIGH |
| Coverage dropped < 2% but consistently over multiple PRs | Slow erosion, likely no enforcement | MEDIUM |
| Test files deleted without corresponding source deletion | Tests removed intentionally to hide failures | CRITICAL |
| Coverage tool configuration changed to exclude more files | May be legitimate or may be gaming the metric | MEDIUM |

**Detection pattern -- coverage gaming**:

```python
# Suspicious: coverage exclusion added in the same PR as new code
# Before (in .coveragerc):
omit = ["*/tests/*", "*/migrations/*"]

# After (in same PR that adds src/services/payment.py):
omit = ["*/tests/*", "*/migrations/*", "*/services/*"]
#                                       ^^^^^^^^^^^^^^ suspicious
```

```javascript
// Suspicious: test that exists only to inflate coverage
test('coverage padding', () => {
  // Imports every module to mark lines as covered
  require('./moduleA');
  require('./moduleB');
  require('./moduleC');
  // No assertions
});
```

**Severity guide**:

| Condition | Severity |
|-----------|----------|
| Tests deleted without corresponding source code deletion | CRITICAL |
| Coverage exclusion list expanded to hide new untested code | HIGH |
| Coverage dropped >5% with no team discussion or tracking issue | HIGH |
| Coverage dropped 2-5% in a single PR | MEDIUM |
| No coverage regression mechanism in place | MEDIUM |
| Coverage ratchet or diff-coverage enforcement in CI | INFO (positive) |

---

## Quick-Reference Summary

| Check | Key Threshold / Signal | Severity |
|-------|----------------------|----------|
| Line coverage only (no branch) | Presented as sufficient | MEDIUM |
| Tests cover lines but have no assertions | Coverage inflation | HIGH |
| No coverage tool configured | Production project | HIGH |
| Branch coverage not enabled | Coverage tool present | MEDIUM |
| Coverage below project threshold by >20 points | Any project type | HIGH |
| Coverage below project threshold by 10-20 points | Any project type | MEDIUM |
| New code at 0% coverage | Any PR | HIGH |
| Auth/payment code below 95% branch coverage | Critical path | CRITICAL |
| Data mutation code below 90% branch coverage | Critical path | HIGH |
| No integration tests for DB-interacting code | External dependencies | HIGH |
| No E2E tests for auth/payment workflows | Critical workflow | CRITICAL |
| E2E tests disabled due to flakiness | Test infrastructure | HIGH |
| No coverage in CI | Pipeline configuration | HIGH |
| Coverage enforced but no diff-coverage | Regression prevention | MEDIUM |
| Tests deleted without source deletion | Coverage gaming | CRITICAL |
| Coverage exclusions expanded without justification | Coverage gaming | HIGH |
| Error-handling code excluded from coverage | Exclusion quality | HIGH |
| Well-configured coverage pipeline with enforcement | Positive pattern | INFO |
