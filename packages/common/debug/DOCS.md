# @affine/debug — Code Documentation

Namespace-based debug logging for AFFiNE. Wraps the `debug` npm library with structured log levels (debug / info / warn / error) and session-persistent enable/disable control.

---

## Basic usage

```ts
import { DebugLogger } from '@affine/debug';

const logger = new DebugLogger('affine:workspace');

logger.debug('loading workspace', { workspaceId });
logger.info('workspace loaded');
logger.warn('slow network detected');
logger.error('failed to sync', error);
```

Output format: `affine:workspace [DEBUG] loading workspace { workspaceId: 'abc' }`

---

## Namespaces

Namespaces follow the `debug` library convention. Use `:` as separator:

```ts
const base = new DebugLogger('affine:sync');
const child = base.namespace('doc'); // → 'affine:sync:doc'
```

---

## Enabling logs

### In the browser

Logs are disabled by default in production. Enable them:

- **URL parameter:** Add `?debug` to any URL → enables all `affine:*` logs.
- **DevTools console:** `localStorage.setItem('debug', 'affine:sync:*')` → filter by namespace. Follows the `debug` library glob syntax.
- **Session storage:** Setting persists for the browser tab session.

### In Node.js

Set the `DEBUG` environment variable:

```bash
DEBUG=affine:* yarn dev
DEBUG=affine:sync:*,affine:auth yarn dev
```

### Excluding namespaces

Prefix with `-` to exclude:

```bash
DEBUG=affine:*,-affine:livedata
```

---

## API

```ts
class DebugLogger {
  constructor(namespace: string);

  debug(message: string, ...args: unknown[]): void;
  info(message: string, ...args: unknown[]): void;
  warn(message: string, ...args: unknown[]): void;
  error(message: string, ...args: unknown[]): void;

  // Dynamic enable/disable
  get enabled(): boolean;
  set enabled(value: boolean);

  // Create child logger with extended namespace
  namespace(extra: string): DebugLogger;
}
```

---

## When to use which level

| Level   | When                                                                     |
| ------- | ------------------------------------------------------------------------ |
| `debug` | Detailed internals — noisy, only useful when debugging a specific issue. |
| `info`  | Notable lifecycle events — workspace loaded, user signed in.             |
| `warn`  | Something unexpected but recoverable — slow API, retry triggered.        |
| `error` | Actual failures that need attention — sync failed, request errored.      |
