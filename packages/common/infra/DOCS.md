# @toeverything/infra — Code Documentation

The foundation layer for all AFFiNE frontend state management. Provides a custom IoC (Inversion of Control) container, a reactive data primitive (`LiveData`), an ORM over Yjs, and an RPC pattern for Worker communication.

---

## Table of Contents

1. [Why this package exists](#1-why-this-package-exists)
2. [Framework — Dependency Injection](#2-framework--dependency-injection)
3. [Component types: Service, Entity, Store, Scope](#3-component-types-service-entity-store-scope)
4. [LiveData — reactive state](#4-livedata--reactive-state)
5. [ORM — Yjs-backed typed tables](#5-orm--yjs-backed-typed-tables)
6. [Op — typed RPC over Workers](#6-op--typed-rpc-over-workers)
7. [Storage — key-value abstractions](#7-storage--key-value-abstractions)
8. [Utilities](#8-utilities)

---

## 1. Why this package exists

AFFiNE supports 6 app targets (web, Electron, iOS, Android, mobile web, admin). They share the same business logic in `@affine/core`, but inject different implementations (e.g., SQLite vs IndexedDB, desktop window vs mobile nav). A standard DI container handles this cleanly.

React Context alone doesn't work because:

- Services can exist outside React (Workers, background sync)
- Services have lifetimes tied to workspaces/scopes, not component trees
- Services have dependencies on other services that need resolution order

---

## 2. Framework — Dependency Injection

**File:** `src/framework/core/framework.ts`

The `Framework` object is a registry. You register component factories into it. Then a `FrameworkProvider` creates instances from those factories, resolving dependencies automatically.

### Registration

```ts
import { Framework } from '@toeverything/infra';

const framework = new Framework();

// Register a service with its dependencies
framework.service(WorkspacesService, [WorkspaceFlavoursService, AuthService]);

// Register a concrete implementation for an abstract identifier
framework.impl(AbstractStorageService, LocalStorageServiceImpl);

// Register an entity (per-scope instance)
framework.entity(Workspace, [WorkspaceScope]);

// Register a scope
framework.scope(WorkspaceScope);
```

### Wiring into React

```tsx
import { FrameworkStackProvider } from '@toeverything/infra';

<FrameworkStackProvider framework={framework}>
  <App />
</FrameworkStackProvider>;
```

### Consuming in React

```ts
import { useService, useServices } from '@toeverything/infra';

const workspacesService = useService(WorkspacesService);
const { authService, configService } = useServices({ AuthService, ConfigService });
```

### Key methods on `Framework`

| Method                                          | What it does                                |
| ----------------------------------------------- | ------------------------------------------- |
| `framework.service(Class, deps[])`              | Register a singleton service                |
| `framework.entity(Class, deps[])`               | Register an entity (instantiated per scope) |
| `framework.store(Class, deps[])`                | Register a store                            |
| `framework.scope(Class)`                        | Register a scope container                  |
| `framework.impl(Identifier, Class, deps[])`     | Bind abstract → concrete                    |
| `framework.override(Identifier, Class, deps[])` | Replace an existing binding (for tests)     |

### Identifier system

Instead of using class references directly (which break with minification), dependencies are declared via `createIdentifier`:

```ts
// Declaration
const AbstractStorage = createIdentifier<StorageInterface>('AbstractStorage');

// Binding
framework.impl(AbstractStorage, ConcreteStorageImpl);

// Injection
class MyService {
  constructor(@Inject(AbstractStorage) private storage: StorageInterface) {}
}
```

---

## 3. Component types: Service, Entity, Store, Scope

All components extend one of these base classes from `src/framework/core/components/`.

### `Service`

Singleton within a scope. Created once, lives until scope is destroyed.

```ts
class AuthService extends Service {
  constructor(private readonly config: Config) {
    super();
  }

  signIn(email: string, password: string) { ... }
}
```

### `Entity<Props>`

Created fresh for each instance. Multiple instances can coexist (e.g., one `Workspace` entity per open workspace).

```ts
class Workspace extends Entity<{ workspaceId: string }> {
  readonly id = this.props.workspaceId;
  readonly docs$ = new LiveData<Doc[]>([]);
}
```

### `Store`

Data source — wraps external storage and exposes reactive data. Usually used with the ORM system.

```ts
class WorkspaceStore extends Store {
  constructor(private readonly db: WorkspaceDB) {
    super();
  }
  // exposes LiveData backed by the ORM
}
```

### `Scope`

Context container. When a scope is created, all entities/stores registered under it are available inside it. Creating a `WorkspaceScope` gives access to workspace-specific services.

```ts
// Creating a scope programmatically
const workspaceScope = framework.createScope(WorkspaceScope, { workspaceId: 'abc' });
const docsService = workspaceScope.get(DocsService);
```

### React lifecycle hook: `useServiceOptional`

Returns `null` instead of throwing when the service isn't registered — useful for optional features.

---

## 4. LiveData — reactive state

**File:** `src/livedata/livedata.ts`

`LiveData<T>` is the primary reactive primitive. It extends RxJS `Observable<T>` but always replays the latest value to new subscribers (like `BehaviorSubject`).

### Basic usage

```ts
const count$ = new LiveData(0);

count$.next(1);              // update
console.log(count$.value);   // read synchronously

count$.subscribe(v => { ... }); // subscribe (immediately gets current value)
```

### Derived LiveData

```ts
// From another LiveData
const doubled$ = count$.map(v => v * 2); // NOT a LiveData
// Use LiveData.computed for derivation that stays LiveData:
const doubled$ = LiveData.computed(get => get(count$) * 2);

// From any Observable
const fromApi$ = LiveData.from(
  apiObservable$,
  initialValue // required — LiveData must always have a value
);

// From Preact signal
const fromSignal$ = LiveData.fromSignal(mySignal);
```

### React hook

```ts
// Re-renders when value changes
const count = useLiveData(count$);

// With selector — only re-renders when selector output changes
const isLoggedIn = useLiveData(auth$, s => s !== null);
```

### `effect()` — side effects

```ts
// Like useEffect but for LiveData, runs outside React
const cleanup = effect(
  map(() => authService.currentUser$),
  switchMap(user => (user ? loadProfile(user.id) : EMPTY))
);
// returns unsubscribe fn
```

### Custom RxJS operators in this package

| Operator                       | Purpose                             |
| ------------------------------ | ----------------------------------- |
| `backoffRetry(options)`        | Retry with exponential backoff      |
| `smartRetry(options)`          | Retry only on retryable errors      |
| `exhaustMapSwitchUntilChanged` | Drop queued work when input changes |
| `mapInto(ctor)`                | Map value into a new class instance |
| `throttleUntilChanged(ms)`     | Throttle but emit on change         |

---

## 5. ORM — Yjs-backed typed tables

**Files:** `src/orm/`

A lightweight ORM where tables live inside a Yjs document. All changes are CRDTs — multiple clients can modify the same table concurrently and changes merge automatically.

### Defining a schema

```ts
import { createORMClient, Table, f } from '@toeverything/infra';

const WorkspaceSchema = {
  tags: Table('tags', {
    id: f.string().primaryKey(),
    name: f.string(),
    color: f.string().optional(),
  }),
  favorites: Table('favorites', {
    id: f.string().primaryKey(),
    target: f.string(),
    index: f.string().optional(),
  }),
};
```

### Creating a client

```ts
const db = createORMClient(WorkspaceSchema);

// connect to a Yjs adapter
db.connect(new YjsDBAdapter(yjsDoc, WorkspaceSchema));
```

### Querying and mutating

```ts
// Insert or update
db.tags.create({ id: nanoid(), name: 'Important', color: '#ff0000' });

// Find
const tag = db.tags.find('tag-id-123');
const all = db.tags.findAll();

// Reactive query — returns Observable
const tags$ = db.tags.find$('tag-id-123');
const all$ = db.tags.findAll$();

// Update
db.tags.update('tag-id-123', { color: '#00ff00' });

// Delete
db.tags.delete('tag-id-123');
```

### Yjs adapter

`YjsDBAdapter` stores each table as a Yjs `YMap<YMap<any>>`. The outer map is keyed by primary key; each inner map is a row. This means row-level updates are independent CRDTs — concurrent edits to different rows never conflict.

---

## 6. Op — typed RPC over Workers

**Files:** `src/op/`

A typed request/response + subscription protocol for communicating across `Worker`, `SharedWorker`, `BroadcastChannel`, or `MessageChannel` boundaries.

### Defining operations

```ts
import { Op } from '@toeverything/infra';

// Operation schema: name → { input, output }
interface StorageOps {
  'storage:get': Op<{ key: string }, string | null>;
  'storage:set': Op<{ key: string; value: string }, void>;
  'storage:watch': Op<{ key: string }, string | null>; // Observable output
}
```

### Consumer (Worker side)

```ts
import { OpConsumer } from '@toeverything/infra';

const consumer = new OpConsumer<StorageOps>(self /* worker globalThis */);

consumer.register('storage:get', async ({ key }) => {
  return localStorage.getItem(key);
});

// Observable ops — for streaming/push
consumer.register('storage:watch', ({ key }) => {
  return new Observable(subscriber => {
    const handler = () => subscriber.next(localStorage.getItem(key));
    window.addEventListener('storage', handler);
    return () => window.removeEventListener('storage', handler);
  });
});

consumer.listen();
```

### Client (main thread side)

```ts
import { OpClient } from '@toeverything/infra';

const worker = new Worker('./storage.worker.js');
const client = new OpClient<StorageOps>(worker);

// One-shot call
const value = await client.call('storage:get', { key: 'theme' });

// Subscription
const sub = client.subscribe('storage:watch', { key: 'theme' }).subscribe(v => {
  console.log('theme changed:', v);
});
```

### Transport support

`OpClient` / `OpConsumer` accept any object with `postMessage` / `addEventListener('message', ...)`. Supported:

- `Worker` (dedicated)
- `SharedWorker.port`
- `MessageChannel.port1/port2`
- `BroadcastChannel`

The `transfer()` wrapper marks `Transferable` objects (ArrayBuffer, MessagePort) for zero-copy transfer:

```ts
await client.call('storage:set', transfer({ key: 'data', value: buffer }, [buffer]));
```

---

## 7. Storage — key-value abstractions

**Files:** `src/storage/`

Simple `Memento` interface for synchronous key-value storage (similar to Android's `SharedPreferences`):

```ts
interface Memento {
  get<T>(key: string): T | undefined;
  set(key: string, value: unknown): void;
  del(key: string): void;
  keys(): string[];
}
```

`AsyncMemento` is the async variant. Implementations provided:

- `LocalStorageMemento` — wraps `localStorage`
- `SessionStorageMemento` — wraps `sessionStorage`
- `MemoryMemento` — in-memory (used in tests)

Used by services that need to persist small configuration values without going through the full ORM.

---

## 8. Utilities

**Files:** `src/utils/`

| Export                    | Description                                                      |
| ------------------------- | ---------------------------------------------------------------- |
| `AsyncLock`               | Async mutex — queues callers, releases one at a time             |
| `AsyncQueue<T>`           | Async FIFO queue with `push(item)` and `next(): Promise<T>`      |
| `throwIfAborted(signal)`  | Throws `AbortError` if signal is aborted                         |
| `MANUALLY_STOP`           | Sentinel error to distinguish user-initiated stops from failures |
| `mergeUpdates(updates[])` | Merge Yjs update binaries using `yjs.mergeUpdates`               |
| `stableHash(value)`       | Deterministic hash for any JSON-serializable value               |
| `objectPool(factory)`     | Pool of reusable objects with acquire/release                    |
| `fractionalIndexing`      | Generate ordering keys between existing keys                     |
| `exhaustMapWithTrailing`  | Like `exhaustMap` but always runs the trailing emission          |
| `shallowEqual(a, b)`      | Shallow object comparison for LiveData selectors                 |
