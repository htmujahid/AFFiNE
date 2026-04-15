# affine_mobile_native — Code Documentation

A Rust library compiled via UniFFI to Swift (iOS) and Kotlin (Android) bindings. Provides document storage, blob management, full-text search, and diagram preview for the mobile apps — the mobile equivalent of `@affine/native`.

---

## Table of Contents

1. [What UniFFI does](#1-what-uniffi-does)
2. [Storage API](#2-storage-api)
3. [Doc operations](#3-doc-operations)
4. [Blob operations](#4-blob-operations)
5. [Full-text search](#5-full-text-search)
6. [Peer sync](#6-peer-sync)
7. [Preview rendering](#7-preview-rendering)
8. [Hashcash](#8-hashcash)
9. [FFI type encoding](#9-ffi-type-encoding)

---

## 1. What UniFFI does

UniFFI reads the Rust source and generates:

- Swift bindings → used by the iOS Capacitor plugin
- Kotlin bindings → used by the Android Capacitor plugin

The generated bindings are type-safe and async where needed (via `tokio` runtime on the Rust side). The Capacitor layer bridges these native bindings back to JavaScript/TypeScript.

---

## 2. Storage API

The storage pool manages per-workspace SQLite databases. Functions are namespaced by their domain.

### Opening a workspace

```swift
// iOS (Swift)
try await storageConnect(universalId: "local:workspace-id", path: dbPath)
storageDisconnect(universalId: "local:workspace-id")
```

```kotlin
// Android (Kotlin)
storageConnect(universalId = "local:workspace-id", path = dbPath)
storageDisconnect(universalId = "local:workspace-id")
```

---

## 3. Doc operations

```swift
// Push a Yjs update (from Capacitor JS layer)
try await storagePushUpdate(universalId: id, docId: docId, data: base64UpdateString)

// Get latest doc snapshot
let record: DocRecord? = try await storageGetDocSnapshot(universalId: id, docId: docId)
// record.bin — base64 encoded Yjs snapshot binary

// List all doc timestamps
let clocks: [DocClock] = try await storageGetDocTimestamps(universalId: id, after: nil)

// Delete a doc
try await storageDeleteDoc(universalId: id, docId: docId)
```

---

## 4. Blob operations

```swift
// Store a blob
try await storageSetBlob(universalId: id, blob: SetBlob(
  key: "content-hash-abc",
  data: base64ImageString,
  mime: "image/png"
))

// Get a blob
let blob: Blob? = try await storageGetBlob(universalId: id, key: "content-hash-abc")

// List all blobs
let blobs: [ListedBlob] = try await storageListBlobs(universalId: id)

// Delete
try await storageDeleteBlob(universalId: id, key: "content-hash-abc", permanently: false)
try await storageReleaseBlobs(universalId: id)  // hard-delete soft-deleted blobs
```

---

## 5. Full-text search

```swift
// Index a doc
try await storageFtsAddDocument(universalId: id, info: BlockInfo(
  blockId: "block-123",
  flavour: "affine:paragraph",
  content: ["The text content"],
  blob: [],
  refDocId: [],
  refInfo: [],
  parentFlavour: nil,
  parentBlockId: nil,
  additional: nil
))

// Search
let hits: [SearchHit] = try await storageFtsSearch(universalId: id, query: "search term")
// hit.id (blockId), hit.score, hit.terms

// Get highlight ranges
let matches: [MatchRange] = try await storageFtsGetMatches(
  universalId: id, query: "search term", blockId: "block-123"
)
// match.start, match.end (character offsets)

// Crawl a doc for index building
let result: CrawlResult = try await storageCrawlDocData(universalId: id, docId: docId)
// result.blocks, result.title, result.summary
```

---

## 6. Peer sync

Track what remote sync peers have received:

```swift
try await storageSetPeerRemoteClock(universalId: id, peer: peerId, docId: docId, clock: timestamp)
let clock: DocClock? = try await storageGetPeerRemoteClock(universalId: id, peer: peerId, docId: docId)
let pulledAt: Int64? = try await storageGetPeerPulledRemoteVersion(universalId: id, peer: peerId)
try await storageSetPeerPulledRemoteVersion(universalId: id, peer: peerId, version: timestamp)
```

---

## 7. Preview rendering (iOS/Android only)

Mobile-specific feature for rendering diagram previews:

```swift
// Mermaid diagram → SVG
let svg: String = try renderMermaidPreviewSvg(
  code: "graph TD; A-->B;",
  options: MermaidRenderOptions(theme: "default", fontFamily: nil, fontSize: nil)
)

// Typst document → SVG
let svg: String = try renderTypstPreviewSvg(
  code: "#set text(size: 12pt); Hello!",
  options: TypstRenderOptions(fontUrls: [], fontDirs: [])
)
```

---

## 8. Hashcash

Anti-abuse proof-of-work (same algorithm as server):

```swift
let response: String = hashcashMint(resource: "server-challenge", bits: 20)
```

---

## 9. FFI type encoding

All binary data (`Uint8Array` in JS / `ByteArray` in Kotlin / `Data` in Swift) is transmitted as **base64 strings** across the FFI boundary. This is handled transparently by `payload_codec.rs`.

```rust
// FFI types in ffi_types.rs use base64 strings, not raw bytes
pub struct DocRecord {
  pub doc_id: String,
  pub bin: String,       // base64-encoded Yjs binary
  pub timestamp: i64,    // milliseconds since epoch
}

// Conversion: NbDocRecord (raw bytes) → DocRecord (base64)
impl From<NbDocRecord> for DocRecord {
  fn from(record: NbDocRecord) -> Self {
    DocRecord {
      doc_id: record.doc_id,
      bin: BASE64.encode(&record.bin),
      timestamp: record.timestamp.timestamp_millis(),
    }
  }
}
```

The Capacitor plugin layer decodes base64 back to `Uint8Array` before passing to JavaScript.
