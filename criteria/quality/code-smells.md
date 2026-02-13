# Code Smells Reference

This reference provides detailed detection criteria for common code smells. It complements the inline criteria in `review-quality.md` with expanded definitions, heuristics, threshold guidance, and before/after examples.

The `quality-reviewer` agent uses this file as a lookup when evaluating code. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Size Smells

### Long Methods / Functions

**Definition**: A function or method that does too much, making it hard to understand, test, and modify in isolation.

**Detection heuristic**: Count non-blank, non-comment source lines within the function body. Assess whether the function handles multiple distinct concerns (validation, transformation, I/O, notification, etc.).

**Thresholds**:

| Lines (non-blank) | Severity | Condition |
|---|---|---|
| >50 | MEDIUM | Multiple concerns are mixed within the body |
| >50 | INFO | Single clear concern, linear logic (e.g., a state machine or parser) |
| >100 | HIGH | Almost always indicates multiple responsibilities |

**Example**:

Before (smell):
```python
def process_order(order_data):
    # Validate (lines 1-20)
    if not order_data.get("customer_id"):
        raise ValueError("Missing customer_id")
    if not order_data.get("items"):
        raise ValueError("Missing items")
    for item in order_data["items"]:
        if item["quantity"] <= 0:
            raise ValueError("Invalid quantity")
    # ... more validation ...

    # Transform (lines 21-45)
    total = sum(i["price"] * i["quantity"] for i in order_data["items"])
    tax = total * 0.08
    # ... more transformation ...

    # Persist (lines 46-65)
    db.orders.insert({"total": total, "tax": tax, ...})

    # Notify (lines 66-80)
    send_email(order_data["customer_id"], "Order confirmed", ...)
```

After (refactored):
```python
def process_order(order_data):
    validate_order(order_data)
    order = build_order(order_data)
    save_order(order)
    notify_customer(order)
```

---

### Large Classes / Modules

**Definition**: A class or module that accumulates too many responsibilities, becoming a maintenance bottleneck where unrelated changes collide.

**Detection heuristic**: Count total lines, public methods, and direct dependency imports. Check for vague naming that resists specificity.

**Thresholds**:

| Metric | Threshold | Severity |
|---|---|---|
| >300 lines | MEDIUM | Review for split opportunities |
| >500 lines | HIGH | Almost certainly multiple responsibilities |
| >20 methods | HIGH | Too many behaviors in one type |
| >10 dependency imports | MEDIUM | Suggests broad coupling |
| Vague names ("Manager", "Helper", "Util", "Service" without qualifier) | MEDIUM | Naming indicates unclear responsibility |

**Example**:

Before (smell):
```java
public class OrderManager {
    // 450 lines, 25 methods
    public void createOrder() { ... }
    public void validateOrder() { ... }
    public void calculateTax() { ... }
    public void applyDiscount() { ... }
    public void sendConfirmationEmail() { ... }
    public void generateInvoicePdf() { ... }
    public void syncToErp() { ... }
    public void updateInventory() { ... }
    // ... 17 more methods
}
```

After (refactored):
```java
public class OrderCreator { ... }          // creation + validation
public class OrderPricing { ... }          // tax, discounts
public class OrderNotifier { ... }         // email, PDF
public class InventoryAdjuster { ... }     // stock updates
```

---

### Long Parameter Lists

**Definition**: A function that takes so many parameters that call sites become unreadable and fragile. Parameter order mistakes are easy to make and hard to spot.

**Detection heuristic**: Count parameters in the function signature. Exclude `self`/`this`/`cls` in object-oriented languages. Consider whether parameters form natural groups.

**Thresholds**:

| Count | Severity | Notes |
|---|---|---|
| >5 | MEDIUM | Consider a parameter object or struct |
| >8 | HIGH | Strong signal of design problems |
| Builder/config constructors | Exception | Many parameters can be acceptable when the function's purpose is construction |

**Example**:

Before (smell):
```python
def create_user(first_name, last_name, email, phone, street, city,
                state, zip_code, country, role, department, manager_id):
    ...
```

After (refactored):
```python
@dataclass
class UserProfile:
    first_name: str
    last_name: str
    email: str
    phone: str

@dataclass
class Address:
    street: str
    city: str
    state: str
    zip_code: str
    country: str

def create_user(profile: UserProfile, address: Address,
                role: str, department: str, manager_id: str):
    ...
```

---

### Deep Nesting

**Definition**: Code with many levels of indentation from nested control structures, making the logical flow difficult to follow and the conditions under which code executes hard to reason about.

**Detection heuristic**: Count nesting depth from `if`, `for`, `while`, `switch`/`match`, `try`/`catch`, callbacks, and closures. Each level adds one to the depth.

**Thresholds**:

| Depth | Severity | Notes |
|---|---|---|
| >3 levels | MEDIUM | Consider early returns, guard clauses, or extraction |
| >5 levels | HIGH | Serious readability problem |

**Example**:

Before (smell):
```javascript
function processItems(items) {
    if (items) {                                          // level 1
        for (const item of items) {                       // level 2
            if (item.isActive) {                          // level 3
                if (item.quantity > 0) {                  // level 4
                    try {                                 // level 5
                        if (item.needsApproval) {         // level 6
                            // actual logic buried here
                        }
                    } catch (e) { ... }
                }
            }
        }
    }
}
```

After (refactored):
```javascript
function processItems(items) {
    if (!items) return;

    for (const item of items) {
        if (!item.isActive || item.quantity <= 0) continue;
        processItem(item);
    }
}

function processItem(item) {
    try {
        if (!item.needsApproval) return;
        // actual logic here, at manageable depth
    } catch (e) { ... }
}
```

---

## 2. Naming Smells

### Single-Letter Variables

**Definition**: Variables named with a single character, providing no context about what the value represents.

**Detection heuristic**: Flag single-letter variable names outside of established conventions. Accepted exceptions: `i`, `j`, `k` for loop counters; `x`, `y`, `z` for coordinates; `e` for exceptions in short catch blocks; `_` for intentionally ignored values; single-letter type parameters in generics (`T`, `K`, `V`).

**Threshold**: MEDIUM when the variable has a non-trivial scope or is used more than 2-3 lines from its declaration.

**Example**:

Before (smell):
```python
def calculate(d, r, t):
    a = d * (1 + r) ** t
    return a
```

After (refactored):
```python
def calculate_compound_amount(principal, annual_rate, years):
    future_value = principal * (1 + annual_rate) ** years
    return future_value
```

---

### Misleading Names

**Definition**: Names that suggest a different behavior, type, or purpose than what the code actually implements. These are worse than vague names because they actively create false mental models.

**Detection heuristic**: Compare what the name implies (verb suggests action, noun suggests data type, prefix/suffix suggests pattern) against what the implementation actually does. Flag contradictions.

**Threshold**: HIGH. Misleading names cause bugs because developers trust the name and skip reading the implementation.

**Example**:

Before (smell):
```python
def validate_email(email):
    """This function doesn't validate -- it normalizes."""
    return email.strip().lower()

def get_users():
    """Name suggests a read, but it also deletes expired users."""
    db.users.delete_many({"expired": True})
    return db.users.find()
```

After (refactored):
```python
def normalize_email(email):
    return email.strip().lower()

def purge_expired_and_get_users():
    db.users.delete_many({"expired": True})
    return db.users.find()
```

---

### Inconsistent Conventions

**Definition**: Mixing naming conventions within the same module or codebase without a language-driven reason.

**Detection heuristic**: Check for mixed casing styles (`camelCase` vs. `snake_case` vs. `PascalCase`) within the same file. Distinguish from language conventions (e.g., Python classes are PascalCase, functions are snake_case -- that is correct). Flag cases where the same type of identifier uses different styles.

**Threshold**: LOW. This is a style issue but accumulates into cognitive friction.

**Example**:

Before (smell):
```python
def getUserData():      # camelCase
    pass

def process_order():    # snake_case
    pass

def ValidateInput():    # PascalCase (not a class)
    pass
```

After (refactored):
```python
def get_user_data():
    pass

def process_order():
    pass

def validate_input():
    pass
```

---

### Poor Boolean Naming

**Definition**: Boolean variables and functions that do not read as predicates, making conditional logic harder to parse at a glance.

**Detection heuristic**: Boolean variables and return values should use prefixes like `is_`, `has_`, `can_`, `should_`, `was_`, `will_`, or read naturally as yes/no questions.

**Threshold**: LOW. Minor readability improvement but meaningful in aggregate.

**Example**:

Before (smell):
```python
active = True
permission = check_admin(user)
if data and valid:
    ...
```

After (refactored):
```python
is_active = True
has_admin_permission = check_admin(user)
if has_data and is_valid:
    ...
```

---

## 3. Duplication Smells

### Copy-Paste Code

**Definition**: Identical or near-identical code blocks appearing in multiple locations. Changes to the logic must be applied to all copies, creating divergence risk.

**Detection heuristic**: Look for code blocks of 5+ lines that appear in more than one location with only whitespace, variable name, or literal value differences. Pay attention to similar structure and flow.

**Thresholds**:

| Copies | Severity |
|---|---|
| 2 copies | MEDIUM |
| 3+ copies | HIGH |

**Example**:

Before (smell):
```python
# In payment_client.py
for attempt in range(3):
    try:
        return self._post(url, payload)
    except ConnectionError:
        time.sleep(2 ** attempt)
raise ConnectionError("Failed after 3 retries")

# In shipping_client.py (near-identical)
for attempt in range(3):
    try:
        return self._post(url, payload)
    except ConnectionError:
        time.sleep(2 ** attempt)
raise ConnectionError("Failed after 3 retries")
```

After (refactored):
```python
# In shared/retry.py
def retry_with_backoff(fn, max_attempts=3, backoff_base=2):
    for attempt in range(max_attempts):
        try:
            return fn()
        except ConnectionError:
            if attempt == max_attempts - 1:
                raise
            time.sleep(backoff_base ** attempt)

# In payment_client.py
return retry_with_backoff(lambda: self._post(url, payload))
```

---

### Similar Logic with Minor Variations

**Definition**: Code blocks that follow the same structure and purpose but differ in small details (field names, thresholds, format strings). These are duplicates that were not recognized as such because the differences obscure the shared pattern.

**Detection heuristic**: Look for functions or blocks that have the same control flow but operate on different fields, use different constants, or vary in only one or two expressions. Often appears in CRUD operations, serializers, validators, or report generators.

**Threshold**: MEDIUM. Harder to spot than exact duplicates but equally dangerous for divergence.

**Example**:

Before (smell):
```python
def validate_shipping_address(addr):
    errors = []
    if not addr.get("street"):
        errors.append("street is required")
    if not addr.get("city"):
        errors.append("city is required")
    if not addr.get("zip") or len(addr["zip"]) != 5:
        errors.append("valid zip is required")
    return errors

def validate_billing_address(addr):
    errors = []
    if not addr.get("street"):
        errors.append("street is required")
    if not addr.get("city"):
        errors.append("city is required")
    if not addr.get("zip") or len(addr["zip"]) != 5:
        errors.append("valid zip is required")
    if not addr.get("name_on_card"):
        errors.append("name on card is required")
    return errors
```

After (refactored):
```python
def validate_address(addr, required_fields=None):
    base_fields = ["street", "city", "zip"]
    all_fields = base_fields + (required_fields or [])
    errors = []
    for field in all_fields:
        if not addr.get(field):
            errors.append(f"{field} is required")
    if addr.get("zip") and len(addr["zip"]) != 5:
        errors.append("valid zip is required")
    return errors

# Usage
validate_address(addr)                                    # shipping
validate_address(addr, required_fields=["name_on_card"])  # billing
```

---

### Repeated Magic Values

**Definition**: The same literal value (number or string) used in multiple places throughout the codebase, creating a hidden dependency. When the value needs to change, all occurrences must be found and updated.

**Detection heuristic**: Flag numeric or string literals that appear in logic (not declarations) more than once across different functions or files. Exclude universally understood values: `0`, `1`, `-1`, `""`, `true`, `false`, `null`/`None`.

**Thresholds**:

| Occurrences | Severity |
|---|---|
| Same value in 2 locations | LOW |
| Same value in 3+ locations | MEDIUM |
| Same value in multiple files | HIGH |

**Example**:

Before (smell):
```python
# In scheduler.py
time.sleep(86400)

# In cache.py
redis.expire(key, 86400)

# In reporting.py
cutoff = datetime.now() - timedelta(seconds=86400)
```

After (refactored):
```python
# In constants.py
SECONDS_PER_DAY = 86400

# In scheduler.py
time.sleep(SECONDS_PER_DAY)

# In cache.py
redis.expire(key, SECONDS_PER_DAY)

# In reporting.py
cutoff = datetime.now() - timedelta(seconds=SECONDS_PER_DAY)
```

---

## 4. Dead Code Smells

### Unreachable Code

**Definition**: Code that can never execute because it follows an unconditional `return`, `throw`, `break`, `continue`, `sys.exit()`, or similar control flow terminator.

**Detection heuristic**: Look for statements after unconditional exits. Also check for branches that can never be true given surrounding constraints (e.g., `if x > 10` inside a block where `x` is already guaranteed to be `<= 10`).

**Threshold**: MEDIUM. Dead code misleads readers about what the function does and adds maintenance burden.

**Example**:

Before (smell):
```python
def get_status(code):
    if code == 200:
        return "ok"
    elif code == 404:
        return "not found"
    else:
        return "error"
    logger.info(f"Status checked: {code}")  # unreachable
```

After (refactored):
```python
def get_status(code):
    logger.info(f"Status checked: {code}")
    if code == 200:
        return "ok"
    elif code == 404:
        return "not found"
    else:
        return "error"
```

---

### Unused Imports / Variables / Functions

**Definition**: Identifiers that are declared or imported but never referenced. They clutter the namespace, slow comprehension, and may indicate incomplete refactoring.

**Detection heuristic**: Check for imports not referenced elsewhere in the file. Check for local variables assigned but never read. Check for functions/methods defined but never called from within the module or from external consumers.

**Thresholds**:

| Type | Severity | Notes |
|---|---|---|
| Unused imports | LOW | Trivial to fix, usually caught by linters |
| Unused variables | MEDIUM | May indicate a logic error (intended to use but forgot) |
| Unused functions/classes | MEDIUM | Could indicate incomplete refactoring or dead feature |

**Example**:

Before (smell):
```python
import os
import json           # never used
from datetime import datetime, timedelta  # timedelta never used

def process(data):
    result = transform(data)
    temp = validate(data)  # temp assigned but never read
    return result
```

After (refactored):
```python
import os
from datetime import datetime

def process(data):
    validate(data)
    result = transform(data)
    return result
```

---

### Commented-Out Code Blocks

**Definition**: Source code that has been commented out rather than deleted. Version control exists to preserve history; commented-out code adds noise and creates ambiguity about whether the code should be restored.

**Detection heuristic**: Look for multi-line comments that contain syntactically valid code structures (function calls, assignments, control flow). Distinguish from explanatory comments that happen to include code examples in documentation.

**Thresholds**:

| Size | Severity |
|---|---|
| 1-3 lines | LOW |
| 4-10 lines | LOW |
| >10 lines | MEDIUM |

**Example**:

Before (smell):
```python
def calculate_total(items):
    total = sum(item.price for item in items)
    # old discount logic
    # if customer.is_premium:
    #     discount = total * 0.15
    #     if discount > 50:
    #         discount = 50
    #     total -= discount
    #     logger.info(f"Applied premium discount: {discount}")
    return total
```

After (refactored):
```python
def calculate_total(items):
    total = sum(item.price for item in items)
    return total
```

If the old logic needs to be referenced, it lives in version control history.

---

## 5. Complexity Smells

### Complex Boolean Expressions

**Definition**: Boolean conditions with many terms, especially mixing `and`/`or` without clear grouping, making it hard to reason about when the condition is true.

**Detection heuristic**: Count the number of boolean operators in a single expression. Check for mixed `and`/`or` without explicit parenthetical grouping.

**Thresholds**:

| Conditions | Severity |
|---|---|
| >3 conditions | MEDIUM |
| Mixed `and`/`or` without grouping | MEDIUM |
| >5 conditions | HIGH |

**Example**:

Before (smell):
```python
if user.is_active and user.age >= 18 and not user.is_banned and (
    user.role == "admin" or user.role == "moderator" or
    user.has_permission("post") and user.email_verified):
    allow_post()
```

After (refactored):
```python
is_eligible_age = user.is_active and user.age >= 18 and not user.is_banned
is_privileged_role = user.role in ("admin", "moderator")
has_post_access = user.has_permission("post") and user.email_verified

if is_eligible_age and (is_privileged_role or has_post_access):
    allow_post()
```

---

### Excessive Branching

**Definition**: Functions with many conditional branches (high cyclomatic complexity), making the function hard to test exhaustively and reason about.

**Detection heuristic**: Count distinct execution paths through the function. Each `if`, `elif`/`else if`, `case`, `catch`, `&&`/`||` short-circuit, and ternary adds a path. Also look for long `if/elif` chains that could be dispatch tables or polymorphism.

**Thresholds**:

| Cyclomatic Complexity | Severity |
|---|---|
| >10 | MEDIUM |
| >15 | HIGH |
| >20 | HIGH (strong recommendation to decompose) |

**Example**:

Before (smell):
```python
def get_pricing(product_type, region, customer_tier, is_sale):
    if product_type == "digital":
        if region == "US":
            if customer_tier == "premium":
                if is_sale:
                    return 4.99
                else:
                    return 7.99
            else:
                if is_sale:
                    return 6.99
                else:
                    return 9.99
        elif region == "EU":
            # ... more nesting ...
    elif product_type == "physical":
        # ... more nesting ...
```

After (refactored):
```python
PRICING = {
    ("digital", "US", "premium", True): 4.99,
    ("digital", "US", "premium", False): 7.99,
    ("digital", "US", "standard", True): 6.99,
    ("digital", "US", "standard", False): 9.99,
    # ...
}

def get_pricing(product_type, region, customer_tier, is_sale):
    key = (product_type, region, customer_tier, is_sale)
    price = PRICING.get(key)
    if price is None:
        raise ValueError(f"No pricing defined for {key}")
    return price
```

---

### Feature Envy

**Definition**: A function that accesses another object's data more than its own, suggesting the logic belongs on that other object. This smell indicates misplaced behavior.

**Detection heuristic**: Count references to fields/methods of other objects vs. the function's own object or local state. If more than half of the data accessed belongs to another object, the function likely has feature envy.

**Threshold**: MEDIUM. The logic works but is in the wrong place, making both the host and the envied object harder to change independently.

**Example**:

Before (smell):
```python
class InvoiceGenerator:
    def calculate_line_total(self, order_item):
        base = order_item.price * order_item.quantity
        discount = order_item.discount_rate * base
        tax = (base - discount) * order_item.tax_rate
        return base - discount + tax
```

After (refactored):
```python
class OrderItem:
    def line_total(self):
        base = self.price * self.quantity
        discount = self.discount_rate * base
        tax = (base - discount) * self.tax_rate
        return base - discount + tax

class InvoiceGenerator:
    def calculate_line_total(self, order_item):
        return order_item.line_total()
```

---

## Quick-Reference Summary

| Smell | Key Threshold | Severity |
|---|---|---|
| Long methods | >50 lines (mixed concerns) | MEDIUM |
| Long methods | >100 lines | HIGH |
| Large classes | >300 lines or >20 methods | MEDIUM-HIGH |
| Long parameter lists | >5 params / >8 params | MEDIUM / HIGH |
| Deep nesting | >3 levels / >5 levels | MEDIUM / HIGH |
| Single-letter variables | Non-trivial scope | MEDIUM |
| Misleading names | Name contradicts behavior | HIGH |
| Inconsistent conventions | Mixed styles, same file | LOW |
| Poor boolean naming | Missing predicate prefix | LOW |
| Copy-paste code | 2 copies / 3+ copies | MEDIUM / HIGH |
| Similar logic, minor variations | Parameterizable pattern | MEDIUM |
| Repeated magic values | 3+ occurrences | MEDIUM |
| Unreachable code | After unconditional exit | MEDIUM |
| Unused imports | Not referenced | LOW |
| Unused variables/functions | Declared but not used | MEDIUM |
| Commented-out code | >10 lines | MEDIUM |
| Complex booleans | >3 conditions mixed | MEDIUM |
| Excessive branching | Cyclomatic complexity >15 | HIGH |
| Feature envy | Uses other object's data more than own | MEDIUM |
