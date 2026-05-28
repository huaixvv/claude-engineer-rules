# Backend API Response Envelope Specification

## **Critical**: The rules below apply only to new projects or projects without an existing response envelope. For legacy projects that already have an established response envelope, ignore this spec.

> **Version**: v1.0
> **Scope**: All backend services
> **Reference implementation**: TypeScript (**Critical**: Other languages must implement this spec strictly)

---

## 1. Design Principles

| # | Principle |
|---|-----------|
| 1 | **HTTP status codes belong to HTTP**: Success returns 200; failures use semantic status codes (401/403/404/422/500). Follow REST. |
| 2 | **Unified response envelope**: Always the three-piece set `{ code, msg, data }` |
| 3 | **Business codes are string literals**: e.g. `USER_NOT_FOUND`. Numeric codes (`10001`) are forbidden. |
| 4 | **Errors are dictionarized**: Each project maintains an `Err` dictionary that strongly binds `code + status + msg`. |
| 5 | **Zero magic strings**: Business code references error codes via object references. |
| 6 | **Zero hardcoding**: All responses are derived from the dictionary or default values. |

---

## 2. Wire Protocol (Response Contract)

### 2.1 Response Body Structure

```
{
  code: string    // "0" = success | "USER_NOT_FOUND" or other business error codes
  msg:  string    // Default "success" on success | dictionary message on failure
  data: any       // Business data | error details | null
}
```

### 2.2 Success Response

```
HTTP/1.1 200 OK
{ "code": "0", "msg": "success", "data": <resource | null> }
```

### 2.3 Failure Response

```
HTTP/1.1 <semantic status code>
{ "code": "<error code>", "msg": "<message>", "data": <error details | null> }
```

---

## 3. HTTP Status Code Strategy

| Scenario | HTTP |
|----------|------|
| Business success | 200 |
| Authentication failure (no token / token expired) | 401 |
| Forbidden | 403 |
| Resource not found | 404 |
| Business conflict | 409 |
| Validation failure | 422 |
| Rate limited | 429 |
| Server error / uncaught exception | 500 |

**Rule**: HTTP status codes express only coarse-grained protocol-layer semantics. Fine-grained business semantics are expressed by the `code` field.

---

## 4. CRUD Verb Response Reference

| Method | Scenario | HTTP | `data` |
|--------|----------|------|--------|
| `GET /users` | List | 200 | `Paginated<T>` structure |
| `GET /users/1` | Single resource | 200 | Resource object |
| `POST /users` | Create | 200 | Newly created resource object |
| `PUT /users/1` | Full update | 200 | Updated resource object |
| `PATCH /users/1` | Partial update | 200 | Updated resource object |
| `DELETE /users/1` | Delete | 200 | `null` |

---

## 5. List / Pagination Standard Structure (Mandatory)

Any list query endpoint's `data` must conform to the following structure. Field names are mandatory:

```typescript
interface Paginated<T> {
  list:        T[]              // Data list
  total:       number           // Total count
  nextCursor?: string | null    // For cursor-based pagination (recommended for large datasets)
  page?:       number           // For page-based pagination
  pageSize?:   number           // For page-based pagination
}
```

### Examples

```json
// Cursor-based pagination
{
  "code": "0", "msg": "success",
  "data": { "list": [], "total": 1000, "nextCursor": "abc" }
}

// Page-based pagination
{
  "code": "0", "msg": "success",
  "data": { "list": [], "total": 100, "page": 1, "pageSize": 20 }
}
```

---

## 6. Error Code Dictionary Specification

### 6.1 Naming Rules

ALL_CAPS with underscores.

Examples: `USER_NOT_FOUND`, `ORDER_PAY_FAILED`, `TOKEN_EXPIRED`, `EMAIL_ALREADY_EXISTS`

### 6.2 Field Name = code Value (Mandatory)

The dictionary key must exactly match its `code` string value:

```typescript
// ✅ Correct
USER_NOT_FOUND: { code: "USER_NOT_FOUND", status: 404, msg: "User not found" }

// ❌ Wrong
UserNotFound:   { code: "USER_NOT_FOUND", ... }   // Field name does not match code value
```

**The same rule applies to other languages** (even if it violates the language's default lint conventions, keep the field name exactly equal to the code value in exchange for cross-field correspondence).

### 6.3 Base Dictionary (Required by All Projects)

| code | HTTP | msg |
|------|------|-----|
| `UNAUTHORIZED` | 401 | Unauthorized |
| `TOKEN_EXPIRED` | 401 | Login expired |
| `FORBIDDEN` | 403 | Access denied |
| `RESOURCE_NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource conflict |
| `VALIDATION_FAILED` | 422 | Validation failed |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |

Business-specific error codes are extended by each project as needed.

### 6.4 `VALIDATION_FAILED` data Contract

`data` is an array of field-level errors:

```typescript
data: Array<{
  field: string
  rule:  string
  msg:   string
}>
```

Example:

```json
{
  "code": "VALIDATION_FAILED",
  "msg": "Validation failed",
  "data": [
    { "field": "email", "rule": "format", "msg": "Invalid email format" },
    { "field": "age",   "rule": "min",    "msg": "Age must be at least 18" }
  ]
}
```

---

## 7. Namespace Design

All members are unified under the `Api` namespace and include the following four members:

| Member | Purpose |
|--------|---------|
| `Api.Err` | Error dictionary (object references, zero magic strings) |
| `Api.ok` | Success response factory |
| `Api.fail` | Error throwing factory (creates exception instance) |
| `Api.Error` | Exception class (for `instanceof` checks) |

---

## 8. Structure Definitions

### 8.1 `Spec` (Error Code Schema)

```typescript
interface Spec {
  code:   string
  status: number   // HTTP status code
  msg:    string   // Human-readable message
}
```

### 8.2 `Body<T>` (Response Body)

```typescript
interface Body<T> {
  code: string
  msg:  string
  data: T | null
}
```

### 8.3 `Err` (Error Dictionary)

```typescript
const Err = {
  [fieldName: same as code value]: Spec
}
```

### 8.4 `AppError` (Exception Class)

```typescript
class AppError extends Error {
  code:   string   // From Spec.code
  status: number   // From Spec.status
  msg:    string   // From Spec.msg
  data:   unknown  // Error details injected by the caller
}
```

---

## 9. Interface Signatures (Usage Contract)

### 9.1 Success Factory

```typescript
Api.ok<T>(
  data?: T | null,    // Default null
  code?: string,      // Default "0"
  msg?:  string,      // Default "success"
): Body<T>
```

### 9.2 Error Throwing Factory

```typescript
Api.fail(
  spec: Spec,         // From the Api.Err dictionary
  data?: unknown,     // Default null
): AppError
```

---

## 10. Business Call Patterns

### 10.1 Success Response

```typescript
Api.ok()              // No-data scenarios such as delete
Api.ok(user)          // Most common: return a resource
Api.ok(list)          // List scenarios (list is a Paginated<T> structure)
```

### 10.2 Failure Response

```typescript
throw Api.fail(Api.Err.USER_NOT_FOUND)
throw Api.fail(Api.Err.VALIDATION_FAILED, [
  { field: "email", rule: "format", msg: "Invalid email format" }
])
```

### 10.3 Exception Handling (Middleware Layer)

```typescript
if (err instanceof Api.Error) {
  setHttpStatus(err.status)
  setBody({ code: err.code, msg: err.msg, data: err.data })
} else {
  setHttpStatus(500)
  setBody({ code: "INTERNAL_ERROR", msg: "Internal server error", data: null })
}
```


## 11. Frontend Usage Contract (Reference)

The frontend should implement unified interception (axios interceptors, etc.) based on the Wire Protocol:

```
on success (HTTP 2xx):
  return response.data.data        // Take business data directly

on failure (HTTP 4xx/5xx):
  body = response.data             // { code, msg, data }
  if body.code == "401": logout()
  else if body.code !== 200: toast(body.msg)
  reject(body)
```

---

## 12. Cross-Language Implementation Requirements

When implementing this spec in other languages (Go / Python / Rust, etc.), the following must be guaranteed:

1. **Wire Protocol is fully consistent** (response body field names, HTTP status codes, `code` values)
2. **Namespace is unified** (package/module is named `api`, exposing `Err` / `Ok` / `Fail` / `AppError`)
3. **Dictionary field name = code value**
4. **List pagination field names are consistent** (`list` / `total` / `nextCursor` / `page` / `pageSize`)
5. **Business call style is symmetric**:
   - Success: `api.ok(data)` or equivalent
   - Failure: `api.fail(api.Err.XXX)` or equivalent

Implementation details (such as how to simulate optional parameters) follow each language's idioms.

---
