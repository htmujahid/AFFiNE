# @affine/error — Code Documentation

Typed error handling for AFFiNE. All server errors, network failures, and GraphQL errors are normalised into `UserFriendlyError` so application code can handle them consistently.

---

## Table of Contents

1. [The problem this solves](#1-the-problem-this-solves)
2. [UserFriendlyError](#2-userfriendlyerror)
3. [Checking error types](#3-checking-error-types)
4. [ErrorName — all possible error codes](#4-errorname--all-possible-error-codes)
5. [GraphQLError](#5-graphqlerror)
6. [ErrorData — typed error payloads](#6-errordata--typed-error-payloads)

---

## 1. The problem this solves

Errors in AFFiNE come from multiple sources: GraphQL responses, fetch failures, business logic, timeouts. Without normalisation you'd write:

```ts
// Without this package — brittle
if (e instanceof Error && e.message.includes('not found')) { ... }
if (e?.extensions?.code === 'NOT_FOUND') { ... }
if (e?.status === 404) { ... }
```

With `UserFriendlyError.fromAny()` you write:

```ts
const err = UserFriendlyError.fromAny(e);
if (err.is('WORKSPACE_NOT_FOUND')) { ... }
```

---

## 2. UserFriendlyError

**File:** `src/index.ts`

```ts
class UserFriendlyError extends Error implements UserFriendlyErrorResponse {
  readonly status: number; // HTTP-like status code
  readonly code: string; // machine-readable code (e.g. "WORKSPACE_NOT_FOUND")
  readonly type: string; // error category
  readonly name: ErrorName; // typed union of all error names
  readonly message: string; // human-readable message (from server)
  readonly data?: any; // optional structured payload
  readonly stacktrace?: string; // server stacktrace (only in dev)
}
```

### `UserFriendlyError.fromAny(anything)`

Converts anything to a `UserFriendlyError`:

```ts
try {
  await gql(someQuery, {});
} catch (raw) {
  const err = UserFriendlyError.fromAny(raw);
  // err is always a UserFriendlyError — never rethrows raw
}
```

Conversion rules:

- Already a `UserFriendlyError` → returned as-is.
- `GraphQLError` with `extensions` → wraps the extensions.
- `TypeError: Failed to fetch` → `NETWORK_ERROR`.
- `AbortError` → `REQUEST_ABORTED`.
- Response with status 413 → `CONTENT_TOO_LARGE`.
- Anything else → `INTERNAL_SERVER_ERROR` with the original `.message`.

---

## 3. Checking error types

```ts
const err = UserFriendlyError.fromAny(caughtError);

// Check by name (typed — only valid error names accepted)
err.is('WORKSPACE_NOT_FOUND'); // → boolean
err.is('ACCESS_DENIED');
err.is('NETWORK_ERROR');

// Check by HTTP status
err.isStatus(404); // → boolean
err.isStatus(401);

// Check for network issues (offline, CORS, DNS)
err.isNetworkError(); // → boolean
err.notNetworkError(); // → boolean (convenience)

// Static versions (work on any value)
UserFriendlyError.isNetworkError(e);
UserFriendlyError.notNetworkError(e);
```

---

## 4. ErrorName — all possible error codes

`ErrorName` is a string union. Server-side names come from `@affine/graphql`'s generated `ErrorNames` enum. Client-side additions:

| Code                         | Meaning                                                           |
| ---------------------------- | ----------------------------------------------------------------- |
| `NETWORK_ERROR`              | `TypeError: Failed to fetch`, offline, CORS                       |
| `CONTENT_TOO_LARGE`          | HTTP 413, blob too large                                          |
| `REQUEST_ABORTED`            | `AbortError`, user cancelled                                      |
| _(+ all server error names)_ | e.g. `WORKSPACE_NOT_FOUND`, `ACCESS_DENIED`, `EMAIL_ALREADY_USED` |

---

## 5. GraphQLError

```ts
class GraphQLError extends BaseGraphQLError {
  override extensions: UserFriendlyErrorResponse;
}
```

This extends the base `graphql` library's error with a typed `extensions` field. You normally don't use this directly — `UserFriendlyError.fromAny()` unwraps it.

---

## 6. ErrorData — typed error payloads

Some errors carry structured data. `ErrorData` gives you typed access:

```ts
import type { ErrorData } from '@affine/error';

// Example: EMAIL_TOKEN_NOT_FOUND error has a specific data shape
type EmailTokenData = ErrorData['EMAIL_TOKEN_NOT_FOUND'];

const err = UserFriendlyError.fromAny(caughtError);
if (err.is('EMAIL_TOKEN_NOT_FOUND')) {
  // err.data is typed as EmailTokenData
  console.log(err.data);
}
```

`ErrorData` is a mapped type built by:

1. Taking every `ErrorName` from the server.
2. Finding the matching `*DataType` in `@affine/graphql`'s generated `ErrorDataUnion`.
3. Mapping `SNAKE_CASE` → `PascalCase` to find the right `__typename`.
