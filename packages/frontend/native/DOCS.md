# @affine/native — Code Documentation

NAPI-RS Rust addon for the Electron desktop app. Provides native SQLite document storage, audio capture, diagram rendering (Mermaid/Typst), and cryptographic utilities.

---

## Table of Contents

1. [What this is and how it's loaded](#1-what-this-is-and-how-its-loaded)
2. [DocStoragePool — SQLite document storage](#2-docstoragepool--sqlite-document-storage)
3. [SqliteConnection — low-level SQLite access](#3-sqliteconnection--low-level-sqlite-access)
4. [Audio capture](#4-audio-capture)
5. [Diagram rendering](#5-diagram-rendering)
6. [Hashcash](#6-hashcash)
7. [Full-text search (FTS)](#7-full-text-search-fts)

---

## 1. What this is and how it's loaded

This is a Rust library compiled to a `.node` binary via NAPI-RS. The Electron main/helper process loads it with:

```ts
import native from '@affine/native';
const { DocStoragePool, SqliteConnection, ... } = native;
```

It is **not available in the renderer process**. The renderer calls the helper process via IPC, which then delegates to this native addon.

Supported platforms: macOS (Intel + ARM), Windows (x64 + ARM), Linux (x64 + ARM).

---

## 2. DocStoragePool — SQLite document storage

`DocStoragePool` manages a pool of SQLite databases — one per workspace. It's the primary storage backend for the Electron app.

### Opening a workspace database

```ts
const pool = new DocStoragePool();

// Connect (opens or creates SQLite DB at path)
await pool.connect(universalId, '/path/to/workspace.db');

// Disconnect when workspace is closed
pool.disconnect(universalId);
```

`universalId` is a string like `local:workspace-id` that uniquely identifies the workspace across local/cloud storage.

### Document operations

```ts
// Push a Yjs update (from editor or sync)
await pool.pushUpdate(universalId, docId, updateUint8Array);

// Get the latest merged snapshot
const record: DocRecord | null = await pool.getDocSnapshot(universalId, docId);
// record.bin is a Uint8Array — the merged Yjs document state

// Get pending updates (not yet merged into snapshot)
const updates: DocUpdate[] = await pool.getDocUpdates(universalId, docId);

// List all doc timestamps (for sync)
const clocks: DocClock[] = await pool.getDocTimestamps(universalId);

// Delete a doc
await pool.deleteDoc(universalId, docId);
```

### Blob operations

```ts
// Get a blob by key (content hash)
const blob: Blob | null = await pool.getBlob(universalId, key);
// blob.data is Uint8Array

// Store a blob
await pool.setBlob(universalId, {
  key: 'sha256-abc123',
  data: imageBytes,
  mime: 'image/png',
});

// Delete / list
await pool.deleteBlob(universalId, key);
const all: ListedBlob[] = await pool.listBlobs(universalId);
```

### Peer/sync clocks

```ts
// Track what a remote peer has synced
await pool.setPeerRemoteClock(universalId, peerId, docId, timestamp);
const clock = await pool.getPeerRemoteClock(universalId, peerId, docId);
```

---

## 3. SqliteConnection — low-level SQLite access

`SqliteConnection` provides direct SQLite access for scenarios not covered by `DocStoragePool` (e.g., migration, validation, vacuum).

```ts
const conn = new SqliteConnection('/path/to/database.db');

// Open connection
await conn.connect();

// Check if DB is valid AFFiNE format
const valid: boolean = await conn.validate();

// Schema migration
await conn.migrateAddDocId();

// Vacuuming (compact + defrag)
await conn.checkpoint();
await conn.vacuumInto('/backup/path.db');

// Direct blob access (bypasses pool)
const blob = await conn.getBlob(key);
await conn.addBlob(key, data);
await conn.deleteBlob(key);

// Raw update operations
const updates = await conn.getUpdates(docId);
await conn.insertUpdates(docId, updates);
await conn.replaceUpdates(docId, newUpdates);
```

---

## 4. Audio capture

Used by the AI meeting/recording feature. macOS only (Windows/Linux return empty lists).

### Recording a specific app's audio

```ts
import { ShareableContent, AudioCaptureSession } from '@affine/native';

// List audio-capable applications
const apps: ApplicationInfo[] = await ShareableContent.applications();
// app.processId, app.name, app.bundleIdentifier, app.icon (Buffer)

// Check if a specific app is using the microphone
const isMic: boolean = await ShareableContent.isUsingMicrophone(app.processId);

// Capture audio from a specific app
const session: AudioCaptureSession = await ShareableContent.tapAudio(app.processId, (audioChunk: Float32Array) => {
  // Called with PCM audio data in real-time
  processAudio(audioChunk);
});

// session.sampleRate, session.channels, session.actualSampleRate
session.stop();
```

### Global audio capture (all apps)

```ts
const session = await ShareableContent.tapGlobalAudio(
  [excludeProcessId], // optional: exclude specific PIDs
  chunk => processAudio(chunk)
);
```

### Full recording API (start/stop with file output)

```ts
import { startRecording, stopRecording, abortRecording } from '@affine/native';

const meta = startRecording({
  appProcessId: app.processId, // or omit for global
  excludeProcessIds: [],
  outputDir: '/tmp/recordings',
  format: 'wav', // optional
  sampleRate: 44100, // optional
  channels: 1, // optional
});
// meta.id, meta.filepath, meta.startedAt

// Later...
const artifact = await stopRecording(meta.id);
// artifact.filepath, artifact.durationMs, artifact.size, artifact.degraded
```

### Audio decoding

```ts
import { decodeAudio } from '@affine/native';

const pcmData: Float32Array = await decodeAudio(
  audioFileBuffer,
  44100, // target sample rate (optional)
  'audio.mp3' // filename hint for format detection (optional)
);
```

---

## 5. Diagram rendering

Used to render Mermaid and Typst diagrams to SVG server-side (in the helper process), avoiding the overhead of a browser rendering engine.

```ts
import { renderMermaidSvg, renderTypstSvg } from '@affine/native';

// Mermaid
const result = renderMermaidSvg({
  code: 'graph TD; A-->B; B-->C;',
  options: {
    theme: 'default', // 'default' | 'dark' | 'forest' | 'neutral'
    fontFamily: 'sans-serif',
    fontSize: 16,
  },
});
// result.svg — SVG string

// Typst
const result = renderTypstSvg({
  code: '#set text(font: "New Computer Modern"); Hello, _Typst_!',
  options: {
    fontUrls: [], // URLs of custom fonts
    fontDirs: [], // Filesystem paths of font directories
  },
});
// result.svg — SVG string
```

---

## 6. Hashcash

Server-compatible proof-of-work for anti-abuse (same implementation as `@affine/server-native`):

```ts
import { mintChallengeResponse, verifyChallengeResponse } from '@affine/native';

// Compute a response (runs in a worker thread — async)
const response = await mintChallengeResponse(
  'challenge-string-from-server',
  20 // difficulty bits (optional, default from server)
);

// Verify
const valid = await verifyChallengeResponse(response, 20, 'challenge-string');
```

---

## 7. Full-text search (FTS)

Embedded FTS inside the SQLite databases, powered by SQLite FTS5:

```ts
// Add a document to the search index
await pool.ftsAddDocument(universalId, {
  docId,
  blockId,
  content: 'The extracted text content of this block',
  flavour: 'affine:paragraph',
});

// Search
const hits: NativeSearchHit[] = await pool.ftsSearch(universalId, 'search query');
// hit.id (blockId), hit.score, hit.terms[]

// Get character-level match ranges for highlight
const matches: NativeMatch[] = await pool.ftsGetMatches(universalId, 'search query', blockId);
// match.start, match.end  — character offsets for <mark> highlighting
```

### Content crawling (for index building)

```ts
// Extract structured block data from a Yjs doc binary
const result: NativeCrawlResult = await pool.crawlDocData(universalId, docId);
// result.blocks — NativeBlockInfo[]
// result.title — string
// result.summary — string
```
