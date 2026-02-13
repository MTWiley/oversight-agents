# Inline Finding Format

Use this format when annotating code directly (e.g., in PR review comments, code suggestions, or inline annotations).

## Single Finding

```
**[SEVERITY]** _agent-name_ — **Finding Title**

`path/to/file:42` — Description of the issue.

```language
problematic code
```

**Fix:** Specific fix description.

```language
corrected code
```
```

## Examples

### CRITICAL Finding

```
**[CRITICAL]** _security-reviewer_ — **Hardcoded AWS Access Key**

`src/config/aws.py:15` — AWS access key is hardcoded in source code. This key provides direct access to AWS resources and must be rotated immediately.

```python
AWS_ACCESS_KEY = "AKIAIOSFODNN7EXAMPLE"
```

**Fix:** Use environment variables or AWS IAM roles.

```python
AWS_ACCESS_KEY = os.environ["AWS_ACCESS_KEY_ID"]
```
```

### HIGH Finding

```
**[HIGH]** _quality-reviewer_ — **Swallowed Exception in Payment Handler**

`src/handlers/payment.py:87-92` — Exception is caught and silently ignored in the payment processing path. Failed payments will appear successful to the user.

```python
try:
    process_payment(order)
except Exception:
    pass
```

**Fix:** Log the error and propagate or handle appropriately.

```python
try:
    process_payment(order)
except PaymentError as e:
    logger.error("Payment failed", order_id=order.id, error=str(e))
    raise
```
```

### MEDIUM Finding

```
**[MEDIUM]** _devops-reviewer_ — **Unpinned GitHub Action**

`.github/workflows/ci.yml:23` — Third-party action uses `@main` branch reference instead of a pinned SHA, enabling supply chain attacks.

```yaml
- uses: actions/setup-node@main
```

**Fix:** Pin to a specific commit SHA.

```yaml
- uses: actions/setup-node@1a4442cacd436585916f9831829e205e1e3e12d2  # v4.0.2
```
```

### LOW Finding

```
**[LOW]** _quality-reviewer_ — **Inconsistent Naming Convention**

`src/utils/helpers.js:12` — Function uses camelCase while the rest of the module uses snake_case.

**Fix:** Rename `processData` to `process_data` to match module conventions.
```

## Compact Format (for summary lists)

When space is limited (e.g., PR comment summary):

```
| Severity | File | Finding |
|----------|------|---------|
| CRITICAL | `src/config/aws.py:15` | Hardcoded AWS access key |
| HIGH | `src/handlers/payment.py:87` | Swallowed exception in payment handler |
| MEDIUM | `.github/workflows/ci.yml:23` | Unpinned GitHub Action |
```

## Usage Notes

- Use inline format for PR review comments and code annotations
- Use compact format for summary tables in PR descriptions
- Always include the agent name for traceability
- Always include the file path and line number for navigation
- Include code examples for CRITICAL and HIGH findings
- For MEDIUM and below, code examples are optional if the description is clear
