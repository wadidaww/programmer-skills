# Skill: Coding Standards

Language-agnostic and language-specific coding standards and best practices.

---

## Universal Standards

### Code Organisation
- One public class/type/function per file (except closely related helpers).
- Group files by feature/domain, not by type (avoid `controllers/`, `services/`, `models/` flat directories at the top level).
- Keep modules small and focused; split when a file exceeds ~300 lines.

### Naming Conventions
| Construct | Convention | Example |
|---|---|---|
| Variables | camelCase (JS/TS/Java), snake_case (Python/Rust/Go) | `userCount`, `user_count` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_RETRIES` |
| Classes | PascalCase | `UserRepository` |
| Functions/methods | camelCase (JS/TS/Java), snake_case (Python/Go) | `getUserById` |
| Files | kebab-case (JS/TS), snake_case (Python/Go) | `user-service.ts`, `user_service.py` |
| Interfaces | PascalCase; no `I` prefix | `UserRepository` not `IUserRepository` |

### Function Design
- Maximum function length: 40 lines (warning), 60 lines (hard limit).
- Maximum parameters: 3–4; use a parameter object beyond that.
- Single responsibility: one function, one job.
- No side effects in functions that return values (Command-Query Separation).
- Prefer pure functions; isolate side effects at the edges.

### Error Handling
```typescript
// BAD — error swallowed
try {
  await processOrder(order)
} catch (e) {
  // nothing
}

// BAD — error logged but not rethrown or handled
try {
  await processOrder(order)
} catch (e) {
  console.log(e)
}

// GOOD — error handled with context
try {
  await processOrder(order)
} catch (error) {
  throw new OrderProcessingError(
    `Failed to process order ${order.id}: ${error.message}`,
    { cause: error, orderId: order.id }
  )
}
```

---

## TypeScript / JavaScript

```typescript
// tsconfig.json — required settings
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

- Use `const` for all declarations unless reassignment is needed; never use `var`.
- Use `unknown` instead of `any`; narrow explicitly before use.
- Use discriminated unions to model state:

```typescript
type AsyncResult<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }
```

- Use optional chaining (`?.`) and nullish coalescing (`??`) over manual null checks.
- Use `zod` for runtime validation of untrusted data (API inputs, env vars).
- Avoid barrel files (`index.ts` re-exporting everything); they slow down editors and circular-dep resolution.

---

## Python

```python
# pyproject.toml — required tooling
[tool.ruff]
line-length = 100
select = ["E", "F", "I", "N", "UP", "ANN", "S", "B", "A", "SIM"]

[tool.mypy]
strict = true
```

- Use type hints on all public functions; use `from __future__ import annotations` for forward references.
- Use `dataclasses` or `pydantic` for data containers; avoid raw dicts for structured data.
- Use `pathlib.Path` instead of `os.path`.
- Use f-strings for string formatting; avoid `.format()` and `%` formatting.
- Use context managers (`with`) for resource management.

```python
# BAD
f = open("file.txt")
data = f.read()
f.close()

# GOOD
with open("file.txt") as f:
    data = f.read()
```

---

## Go

```go
// Always handle errors; never discard with _
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)
}

// Use context for cancellation and deadlines
func fetchUser(ctx context.Context, id string) (*User, error) {
    // pass ctx to all downstream calls
    return db.QueryContext(ctx, "SELECT ...", id)
}
```

- Use `errors.Is` and `errors.As` to inspect error chains.
- Use `defer` for cleanup (closing files, unlocking mutexes).
- Use table-driven tests for multiple scenarios.
- Avoid package-level mutable state.

---

## Java / Kotlin

```java
// Use records for immutable value objects (Java 16+)
public record UserId(String value) {
    public UserId {
        Objects.requireNonNull(value, "UserId must not be null");
        if (value.isBlank()) throw new IllegalArgumentException("UserId must not be blank");
    }
}
```

- Use `Optional<T>` instead of returning `null`.
- Use `var` for local type inference where the type is obvious.
- Prefer `List.of()`, `Map.of()` for immutable collections.
- Use `@NotNull` / `@Nullable` annotations on all public API parameters.

---

## Rust

```rust
// Use Result and ? operator for error propagation
fn parse_config(path: &Path) -> Result<Config, ConfigError> {
    let content = fs::read_to_string(path)
        .map_err(|e| ConfigError::IoError { path: path.to_owned(), source: e })?;
    serde_json::from_str(&content)
        .map_err(|e| ConfigError::ParseError { source: e })
}
```

- Use the `newtype` pattern to enforce domain invariants at compile time.
- Use `#[derive(Debug, Clone, PartialEq)]` on data types.
- Use `clippy` (`cargo clippy -- -D warnings`) in CI.
- Document all public items with `///` doc comments.

---

## SQL

- Use `UPPER CASE` for keywords: `SELECT`, `FROM`, `WHERE`, `JOIN`.
- Qualify column names with table aliases in joins.
- Always specify column names in `INSERT`; never use `INSERT INTO table VALUES (...)`.
- Use migrations for all schema changes; never modify the database manually in production.
- Format long queries across multiple lines:

```sql
SELECT
    o.id,
    o.status,
    u.email AS customer_email,
    SUM(oi.quantity * oi.unit_price) AS total
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY o.id, o.status, u.email
ORDER BY o.created_at DESC;
```
