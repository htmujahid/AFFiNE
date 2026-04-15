# @affine/nbstore — Code Documentation

The unified storage abstraction layer for AFFiNE. Every document, blob (image, attachment), and real-time awareness (cursor positions) goes through this layer. It normalises four backends — IndexedDB, SQLite (Electron/mobile), cloud (HTTP+WebSocket), and broadcast channel (IPC) — behind a single interface.

---

## Table of Contents

1. [Core concept](#1-core-concept)
2. [Storage interfaces](#2-storage-interfaces)
3. [SpaceStorage — the composite](#3-spacestorage--the-composite)
4. [Connection lifecycle](#4-connection-lifecycle)
5. [DocStorage — Yjs documents](#5-docstorage--yjs-documents)
6. [BlobStorage — binary attachments](#6-blobstorage--binary-attachments)
7. [AwarenessStorage — presence](#7-awarenessstorage--presence)
8. [IndexerStorage — full-text search](#8-indexerstorage--full-text-search)
9. [Sync system](#9-sync-system)
10. [Backend implementations](#10-backend-implementations)
11. [Worker support](#11-worker-support)
12. [Key data types](#12-key-data-types)

---

## 1. Core concept

AFFiNE is **local-first**: documents live on device first, sync to cloud second. `nbstore` models this with:

- A **local** storage (IndexedDB or SQLite) that is always available.
- Zero or more **remote** storages (cloud, other peers) that sync in the background.
- A **Sync** orchestrator that reconciles local ↔ remote using Yjs state vectors.

```
┌─────────────────────────────────────────────┐
│              SpaceStorage                    │
│  DocStorage  BlobStorage  AwarenessStorage   │
│  IndexerStorage  (+ Sync variants of each)   │
└─────────────┬──────────────────┬────────────┘
              │                  │
          local IDB          remote cloud
          (always on)         (when online)
```

Every storage type has a **local** implementation (fast, offline) and a **sync** companion that talks to remote peers.

---

## 2. Storage interfaces

**File:** `src/storage/storage.ts`

Base interface all storages implement:

```ts
interface Storage {
  readonly storageType: StorageType; // 'doc' | 'blob' | 'awareness' | 'indexer' | ...
  readonly spaceType: SpaceType; // 'workspace' | 'userspace'
  readonly spaceId: string;
  readonly connection: Connection;
  connect(): void;
  disconnect(): void;
}
```

---

## 3. SpaceStorage — the composite

**File:** `src/storage/index.ts`

`SpaceStorage` bundles all storage types for one space (workspace or userspace). You pass it a set of concrete implementations and it manages their connections:

```ts
const storage = new SpaceStorage({
  doc: new IndexedDBDocStorage(spaceId),
  blob: new IndexedDBBlobStorage(spaceId),
  awareness: new AwarenessBroadcastChannelStorage(spaceId),
  indexer: new IndexedDBIndexerStorage(spaceId),
  docSync: new IndexedDBDocSyncStorage(spaceId),
  blobSync: new IndexedDBBlobSyncStorage(spaceId),
  indexerSync: new IndexedDBIndexerSyncStorage(spaceId),
});

storage.connect(); // opens all storage connections
const doc = await storage.get('doc'); // retrieve DocStorage
```

In Electron, you'd swap IndexedDB implementations for SQLite ones. In cloud-only mode, you'd use the cloud implementations. The rest of the application code never changes.

---

## 4. Connection lifecycle

**File:** `src/connection.ts`

Every storage has a `Connection` with status tracked as an Observable:

```ts
type ConnectionStatus = 'idle' | 'connecting' | 'connected' | 'error' | 'closed';

interface Connection {
  readonly status$: Observable<ConnectionStatus>;
  readonly error$: Observable<Error | null>;
  connect(): void;
  disconnect(): void;
  waitForConnected(abort?: AbortSignal): Promise<void>;
}
```

Consumers can react to connection status to show sync indicators:

```ts
storage.doc.connection.status$.subscribe(status => {
  if (status === 'error') showOfflineBanner();
});
```

`SharedConnection` allows multiple storages to share one underlying connection (e.g., one WebSocket for both doc and blob sync).

---

## 5. DocStorage — Yjs documents

**File:** `src/storage/doc.ts`

Documents are stored as **Yjs update binaries** — not JSON, not plain text. A "document" is a sequence of incremental CRDT updates that can be merged into a snapshot.

### Interface

```ts
interface DocStorage extends Storage {
  readonly storageType: 'doc';
  readonly isReadonly: boolean;

  // Get latest merged snapshot for a doc
  getDoc(docId: string): Promise<DocRecord | null>;

  // Get only the updates the caller is missing (using state vector diffing)
  getDocDiff(docId: string, state: Uint8Array): Promise<DocDiff | null>;

  // Push a new update (from user edit or remote peer)
  pushDocUpdate(update: DocUpdate, origin?: string): Promise<DocClock>;

  // List all known docs and their last-updated timestamps
  getDocTimestamps(after?: Date): Promise<DocClocks>;

  // Delete a doc and all its history
  deleteDoc(docId: string): Promise<void>;

  // Subscribe to doc updates from other sources (e.g., another tab)
  subscribeDocUpdate(callback: (update: DocUpdate, origin?: string) => void): () => void;

  // History
  getDocHistory(docId: string, timestamp: Date): Promise<DocRecord | null>;
  listDocHistories(docId: string, options?: { skip?: number; limit?: number }): Promise<DocClock[]>;
  rollbackDoc(docId: string, timestamp: Date): Promise<void>;
}
```

### Key data types

```ts
// A full document snapshot
interface DocRecord {
  docId: string;
  bin: Uint8Array; // merged Yjs update binary
  timestamp: Date;
  editor?: string; // user ID of last editor
}

// Incremental update
interface DocUpdate {
  docId: string;
  bin: Uint8Array; // raw Yjs update binary
  editor?: string;
}

// Diff result — what the caller is missing
interface DocDiff {
  docId: string;
  missing: Uint8Array; // updates the caller doesn't have
  state: Uint8Array; // caller's current state vector
}
```

### How state vector diffing works

Yjs has a built-in protocol for efficient sync:

1. Client sends its **state vector** (`encodeStateVectorFromUpdate`) — a compact summary of what it has.
2. Server calls `getDocDiff(docId, clientStateVector)`, which computes the delta with `diffUpdate()`.
3. Server sends back only the `missing` binary — the client applies it with `Y.applyUpdate()`.

This means syncing a large doc with small changes transfers only the delta, not the entire doc.

---

## 6. BlobStorage — binary attachments

**File:** `src/storage/blob.ts`

Stores binary files (images, PDFs, attachments) keyed by content hash.

```ts
interface BlobStorage extends Storage {
  readonly storageType: 'blob';

  get(key: string): Promise<BlobRecord | null>;
  set(blob: BlobRecord): Promise<void>;
  delete(key: string, permanently?: boolean): Promise<void>;
  list(signal?: AbortSignal): Promise<ListedBlobRecord[]>;
  release(): Promise<void>; // hard-delete soft-deleted blobs
}

interface BlobRecord {
  key: string; // content hash (e.g., SHA256)
  data: Uint8Array;
  mime: string;
  createdAt?: Date;
}

interface ListedBlobRecord {
  key: string;
  mime: string;
  size: number;
  createdAt?: Date;
}
```

Blobs use content-addressed storage: the key is derived from the file content, so identical files are stored only once.

---

## 7. AwarenessStorage — presence

**File:** `src/storage/awareness.ts`

Stores real-time presence data: who is in the document, where their cursor is, what they're selecting. This is purely ephemeral — it's not persisted to disk.

```ts
interface AwarenessStorage extends Storage {
  readonly storageType: 'awareness';

  update(awareness: AwarenessRecord, origin?: string): Promise<void>;
  subscribeUpdate(callback: (awareness: AwarenessRecord, origin?: string) => void): () => void;
}

interface AwarenessRecord {
  spaceId: string;
  awareness: Uint8Array; // encoded Yjs Awareness state
}
```

The `BroadcastChannelAwarenessStorage` implementation is used between browser tabs of the same workspace — a tab change in one tab is immediately reflected in others without a network round-trip.

---

## 8. IndexerStorage — full-text search

**File:** `src/storage/indexer.ts`

Stores extracted block content for full-text search. Separate from `DocStorage` because it stores parsed/processed data, not raw CRDT binaries.

```ts
interface IndexerStorage extends Storage {
  readonly storageType: 'indexer';

  // Store crawled block data for a doc
  setDocIndex(docId: string, crawlResult: CrawlResult): Promise<void>;

  // Fetch indexed data
  getDocIndex(docId: string): Promise<CrawlResult | null>;

  // Full-text search across all docs in the space
  search(query: SearchOptions): Promise<SearchResult[]>;

  // Delete index for a doc
  deleteDocIndex(docId: string): Promise<void>;
}

interface CrawlResult {
  blocks: BlockInfo[];
  title: string;
  summary: string;
}

interface BlockInfo {
  blockId: string;
  flavour: string; // block type: affine:paragraph, affine:heading, etc.
  content?: string[]; // extracted text
  blob?: string[]; // referenced blob keys
  refDocId?: string[]; // referenced doc IDs (links)
  parentFlavour?: string;
  parentBlockId?: string;
}
```

---

## 9. Sync system

**Files:** `src/sync/`

The sync system reconciles local storage with remote peers. It runs continuously in the background.

### Architecture

```
DocSyncImpl
  └── DocSyncPeer[] (one per remote storage)
        ├── pull loop: remote → local
        └── push loop: local → remote
```

### DocSync

```ts
interface DocSync {
  // Overall sync progress
  readonly state$: Observable<DocSyncState>;

  // Per-doc sync status
  docState$(docId: string): Observable<DocSyncDocState>;

  // Wait for a specific doc (or all docs) to be synced
  waitForSynced(docId?: string, abort?: AbortSignal): Promise<void>;

  // Give a doc higher priority (e.g., the currently open doc)
  addPriority(id: string, priority: number): () => void;

  // Restart sync (e.g., after auth token refresh)
  resetSync(): Promise<void>;
}

interface DocSyncState {
  total: number;
  syncing: number;
  synced: boolean;
  retrying: boolean;
  errorMessage: string | null;
}
```

### Sync peer protocol

Each `DocSyncPeer` runs two independent loops:

**Pull loop:**

1. Fetch `remoteTimestamps` (all doc IDs + timestamps from remote).
2. Compare with local timestamps.
3. For docs where remote is newer: fetch remote diff, apply to local.

**Push loop:**

1. Subscribe to local `subscribeDocUpdate`.
2. For each new local update: push to remote.
3. Retry with backoff on network failure.

### Priority system

`addPriority(docId, 100)` puts that doc at the front of the sync queue. The currently-open document gets higher priority so it syncs before background documents.

### BlobSync

Same pattern as DocSync but for blobs. Sync strategy: push any local blob keys that the remote doesn't have; pull any remote keys that local doesn't have.

---

## 10. Backend implementations

**Files:** `src/impls/`

### IndexedDB (`src/impls/idb/`)

Used in browsers. One IDB database per space. Tables:

- `snapshots` — merged Yjs snapshots
- `updates` — pending updates queue
- `blobs` — binary attachments
- `clocks` — sync timestamps

| Class                      | Purpose                 |
| -------------------------- | ----------------------- |
| `IndexedDBDocStorage`      | Full doc storage in IDB |
| `IndexedDBBlobStorage`     | Blob storage in IDB     |
| `IndexedDBDocSyncStorage`  | Sync clock tracking     |
| `IndexedDBBlobSyncStorage` | Blob sync tracking      |
| `IndexedDBIndexerStorage`  | Search index in IDB     |

### SQLite (`src/impls/sqlite/`)

Used in Electron and Capacitor mobile. Talks to the native SQLite via the Op RPC pattern (the actual SQLite operations run in the main process / native layer):

| Class               | Purpose                 |
| ------------------- | ----------------------- |
| `SqliteDocStorage`  | Doc storage via SQLite  |
| `SqliteBlobStorage` | Blob storage via SQLite |

### Cloud (`src/impls/cloud/`)

Communicates with `@affine/server` via:

- GraphQL mutations (for doc operations, blob upload/download)
- WebSocket (real-time doc update push)

### BroadcastChannel (`src/impls/broadcast-channel/`)

`BroadcastChannelAwarenessStorage` — sends awareness updates between browser tabs using `BroadcastChannel` API. No server round-trip needed for cross-tab cursor sync.

---

## 11. Worker support

**Files:** `src/worker/`

`SpaceStorage` can run inside a `SharedWorker` to share one storage connection across all tabs of the same workspace. Uses the `Op` pattern from `@toeverything/infra`:

```ts
// In worker
const consumer = new SpaceStorageWorkerConsumer();
consumer.listen();

// In main thread
const client = new SpaceStorageWorkerClient(sharedWorker.port);
const doc = await client.getDoc('doc-id-123');
```

---

## 12. Key data types

```ts
// Document record (snapshot)
interface DocRecord {
  docId: string;
  bin: Uint8Array; // Yjs encoded state
  timestamp: Date;
  editor?: string;
}

// Document update (delta)
interface DocUpdate {
  docId: string;
  bin: Uint8Array; // Yjs update binary
  editor?: string;
}

// Blob (file attachment)
interface BlobRecord {
  key: string; // content hash
  data: Uint8Array;
  mime: string;
  createdAt?: Date;
}

// Sync state
interface DocSyncState {
  total: number;
  syncing: number;
  synced: boolean;
  retrying: boolean;
  errorMessage: string | null;
}

// Search result block
interface BlockInfo {
  blockId: string;
  flavour: string;
  content?: string[];
  blob?: string[];
  refDocId?: string[];
  parentFlavour?: string;
  parentBlockId?: string;
}
```
