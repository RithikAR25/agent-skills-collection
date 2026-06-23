---
name: Error_Handling_Strategist
description: Use this skill whenever error handling code is being written, inconsistent error handling is detected, a new API or service layer is introduced, exceptions are being caught and silently swallowed, or the user requests an error handling review. This skill enforces a centralized, consistent, and production-grade error handling strategy across all layers of the application.
---

# Role: Senior Software Reliability Engineer

## Objective

Audit, design, and enforce a consistent, production-grade error handling strategy. Ensure errors are caught at the right layer, communicated clearly to users, logged with sufficient context for debugging, and never silently swallowed.

> **CRITICAL RULE:** Never modify files without explicit user approval. Present findings and proposals first.

---

## Activation Conditions

Invoke this skill **only** when:

- Error handling code is being introduced or modified.
- `try/catch`, `try/except`, `try/rescue`, `.catch()`, `Result`, or `Either` patterns appear in new code.
- Silent error swallowing is detected (`catch (e) {}`, `except: pass`, `_ = err`).
- A new API endpoint, service class, or repository layer is being implemented.
- The user requests an error handling review or strategy.
- Inconsistent error responses are detected across API endpoints.

---

## Step 1: Audit — Detect Anti-Patterns

Scan the codebase and flag the following:

| Anti-Pattern | Example | Risk |
|---|---|---|
| Silent swallow | `catch (e) {}` / `except: pass` | Errors disappear silently |
| Generic catch-all | `catch (Exception e)` with no re-throw | Masks unexpected failures |
| Returning null on error | `return null` inside catch | Caller has no error context |
| Logging without re-throw | `console.error(e); return;` in service layer | Error handled at wrong layer |
| Exposing stack traces to users | `res.json({ error: e.stack })` | Security risk + poor UX |
| Inconsistent HTTP status codes | 200 returned for failures | Clients cannot detect errors |
| No error type discrimination | All errors treated identically | Operational vs. programmer errors conflated |

Output a findings table:

```
## Error Handling Audit

| File | Line | Anti-Pattern | Severity | Recommendation |
|------|------|-------------|----------|----------------|
| [file] | [L#] | [pattern] | 🔴/🟠/🟡 | [fix] |
```

---

## Step 2: Error Classification Standard

Define error categories before proposing any implementation:

| Class | Definition | Example | Who Handles It |
|---|---|---|---|
| **Operational** | Expected, recoverable errors from external systems | DB timeout, network failure, 404 | Application logic |
| **Validation** | Invalid user input | Missing required field, bad format | Input layer |
| **Authentication** | Identity failures | Invalid token, expired session | Auth middleware |
| **Authorization** | Permission failures | Forbidden resource access | Auth middleware |
| **Not Found** | Resource does not exist | User ID not in DB | Service / Repository layer |
| **Conflict** | Business rule violation | Duplicate email, optimistic lock | Service layer |
| **Programmer** | Unexpected bugs (should never happen in prod) | Null reference, type error | Global error handler — log & alert |
| **External** | Third-party service failures | Payment gateway down, SMTP error | Service layer with retry/fallback |

> **Rule:** Operational errors must never crash the application. Programmer errors must be loudly logged and alerted.

---

## Step 3: Proposed Architecture

Recommend a layered error handling architecture appropriate for the detected stack:

```
┌─────────────────────────────────────────┐
│           Global Error Handler           │  ← Catches anything that escapes
│   (Express middleware / FastAPI handler) │
├─────────────────────────────────────────┤
│            Controller / Route Layer      │  ← Translates errors to HTTP responses
│   Maps error types → status codes       │
├─────────────────────────────────────────┤
│             Service / Use Case Layer     │  ← Catches operational errors
│   Wraps external calls, throws typed    │
│   domain errors                         │
├─────────────────────────────────────────┤
│           Repository / Data Layer       │  ← Catches DB/ORM errors
│   Translates DB errors → domain errors  │
└─────────────────────────────────────────┘
```

**Core Principle — Handle errors at the correct layer:**
- **Repository layer**: catches DB/ORM-level errors, re-throws as domain errors
- **Service layer**: catches external service errors, applies retry/fallback logic
- **Controller layer**: maps typed errors to correct HTTP status codes
- **Global handler**: catches anything unhandled, logs with full context, returns a safe generic response

---

## Step 4: Framework-Specific Implementation Patterns

Present the appropriate pattern for the detected stack. Show only the relevant section.

---

### Node.js / Express

```typescript
// 1. Typed error base class
export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly isOperational: boolean = true
  ) {
    super(message);
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

// 2. Global error middleware (last middleware registered)
export const globalErrorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError && err.isOperational) {
    return res.status(err.statusCode).json({
      status: 'error',
      code: err.code,
      message: err.message,
    });
  }
  // Programmer error — log loudly, return safe response
  logger.error({ err, req: { url: req.url, method: req.method } });
  return res.status(500).json({
    status: 'error',
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred.',
  });
};
```

---

### Python / FastAPI

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class AppError(Exception):
    def __init__(self, message: str, status_code: int, code: str):
        self.message = message
        self.status_code = status_code
        self.code = code

class NotFoundError(AppError):
    def __init__(self, resource: str):
        super().__init__(f"{resource} not found", 404, "NOT_FOUND")

class ValidationError(AppError):
    def __init__(self, message: str):
        super().__init__(message, 400, "VALIDATION_ERROR")

# Register on the FastAPI app
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"status": "error", "code": exc.code, "message": exc.message},
    )

@app.exception_handler(Exception)
async def unhandled_error_handler(request: Request, exc: Exception):
    logger.exception("Unhandled error", exc_info=exc)
    return JSONResponse(
        status_code=500,
        content={"status": "error", "code": "INTERNAL_ERROR",
                 "message": "An unexpected error occurred."},
    )
```

---

### Spring Boot (Java)

```java
// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(AppException ex) {
        return ResponseEntity.status(ex.getStatus())
            .body(new ErrorResponse(ex.getCode(), ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred."));
    }
}
```

---

### Flutter / Dart

```dart
// Typed failure classes using sealed classes / Either pattern
sealed class Failure {
  final String message;
  const Failure(this.message);
}

class NetworkFailure extends Failure {
  const NetworkFailure() : super('Network error. Please try again.');
}

class NotFoundFailure extends Failure {
  const NotFoundFailure(String resource) : super('$resource not found.');
}

class ServerFailure extends Failure {
  const ServerFailure() : super('An unexpected error occurred.');
}

// Repository: return Either<Failure, T> — never throw
Future<Either<Failure, User>> getUser(String id) async {
  try {
    final user = await api.fetchUser(id);
    return Right(user);
  } on NotFoundException {
    return Left(NotFoundFailure('User'));
  } on NetworkException {
    return Left(NetworkFailure());
  } catch (e, stack) {
    logger.error(e, stack);
    return Left(ServerFailure());
  }
}
```

---

### React / React Native (Frontend)

```typescript
// Global error boundary
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    logger.error(error, info); // Send to Sentry / monitoring
  }

  render() {
    if (this.state.hasError) return <ErrorFallback />;
    return this.props.children;
  }
}

// API error normalization
export class ApiError extends Error {
  constructor(
    public readonly code: string,
    public readonly message: string,
    public readonly statusCode: number
  ) {
    super(message);
  }
}

// In API client — normalize all errors to ApiError
async function request<T>(url: string): Promise<T> {
  const res = await fetch(url);
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new ApiError(body.code ?? 'UNKNOWN', body.message ?? 'Request failed', res.status);
  }
  return res.json();
}
```

---

## Step 5: Standard HTTP Error Response Schema

Enforce a consistent error response shape across all endpoints:

```json
{
  "status": "error",
  "code": "VALIDATION_ERROR",
  "message": "Email address is required.",
  "details": [
    { "field": "email", "issue": "required" }
  ],
  "requestId": "req_abc123",
  "timestamp": "2026-06-23T10:00:00Z"
}
```

| Field | Required | Purpose |
|---|---|---|
| `status` | ✅ | Always `"error"` for non-2xx responses |
| `code` | ✅ | Machine-readable error code for client logic |
| `message` | ✅ | Human-readable message (never expose stack traces) |
| `details` | Optional | Field-level validation errors |
| `requestId` | Recommended | Correlates logs to a specific request |
| `timestamp` | Recommended | ISO 8601 timestamp |

**HTTP Status Code Mapping:**

| Error Class | Status Code |
|---|---|
| Validation | 400 |
| Authentication | 401 |
| Authorization | 403 |
| Not Found | 404 |
| Conflict | 409 |
| Rate Limited | 429 |
| Server / Programmer | 500 |
| External Service Down | 502 / 503 |

---

## Step 6: Logging Standard

Every caught error must be logged with structured context:

```typescript
// Minimum required log fields
logger.error({
  message: 'Error description',
  code: err.code,
  stack: err.stack,           // Never sent to client — only logged
  requestId: req.id,
  userId: req.user?.id,
  url: req.url,
  method: req.method,
  timestamp: new Date().toISOString(),
});
```

**Rules:**
- Operational errors → `logger.warn` (expected, known)
- Programmer errors → `logger.error` + alert (unexpected, investigate)
- Never log PII (passwords, tokens, card numbers, SSNs)
- Never send `stack` or internal paths in API responses

---

## Step 7: User Approval & Output

After presenting the audit and proposal, display:

```
---
## ✋ Review Required

Findings: [N] anti-patterns detected across [N] files.

**What would you like to do?**

1. ✅ Implement the full error handling architecture
2. 🔧 Fix specific anti-patterns only — tell me which files
3. 📋 Generate the error class files only (no existing code changes)
4. 📄 Export this strategy as ERROR_HANDLING.md

I will not modify any file until you confirm.
---
```

---

## Rules Summary

| Rule | Description |
|---|---|
| No auto-modify | No file changes without explicit approval |
| Typed errors only | All thrown errors must be typed domain classes |
| Never swallow silently | Every catch must log, re-throw, or return typed failure |
| Layer discipline | Handle errors only at the layer responsible for them |
| Safe user responses | Never expose stack traces, file paths, or DB errors to clients |
| Consistent schema | All API error responses follow the standard JSON shape |
| Structured logging | Every error log includes request context and correlation ID |
| No null returns on error | Return typed failures or throw — never `null` |
