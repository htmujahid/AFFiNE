# y-octo — Code Documentation

A high-performance, wire-compatible Rust implementation of the Yjs CRDT. Used in server-side doc merging (via `@affine/server-native`) and potentially as a drop-in for Yjs JS in performance-critical paths.

---

## Table of Contents

1. [What is a CRDT and why y-octo?](#1-what-is-a-crdt-and-why-y-octo)
2. [Core types](#2-core-types)
3. [Creating and editing a document](#3-creating-and-editing-a-document)
4. [Sync protocol](#4-sync-protocol)
5. [Awareness](#5-awareness)
6. [History / undo](#6-history--undo)
7. [Wire compatibility with Yjs JS](#7-wire-compatibility-with-yjs-js)
8. [y-octo-utils — tooling](#8-y-octo-utils--tooling)

---

## 1. What is a CRDT and why y-octo?

A **CRDT** (Conflict-free Replicated Data Type) is a data structure that can be updated independently by multiple peers and then merged without conflicts. Yjs is the most widely used CRDT for collaborative text editing.

AFFiNE stores every document as a Yjs CRDT. When two users edit simultaneously, their changes merge automatically. y-octo implements the same Yjs wire format in Rust, which means:

- Binaries produced by y-octo can be applied by Yjs JS, and vice versa.
- Server-side operations (merge, diff, encode) run 10–20× faster than Yjs JS.
- No V8 GC pressure during large-doc operations.

---

## 2. Core types

**File:** `y-octo/core/src/doc/`

```rust
// Root document
pub struct Doc { ... }
pub struct DocOptions {
    pub client_id: u64,
    pub guid: String,
    pub gc: bool,          // garbage collection
}

// Shared types (analogous to Yjs shared types)
pub struct Array  { ... }  // YArray equivalent
pub struct Map    { ... }  // YMap equivalent
pub struct Text   { ... }  // YText equivalent

// State snapshot (what updates have been applied)
pub struct StateVector { ... }

// A single update binary (can be applied to a Doc)
pub struct Update { ... }

// Scalar values stored inside Map/Array
pub enum Value {
    Any(Any),
    Doc(Doc),
    Array(Array),
    Map(Map),
    Text(Text),
}
```

---

## 3. Creating and editing a document

```rust
use y_octo::{Doc, Any};

// Create a document
let doc = Doc::default();

// Get or create a shared type
let text = doc.get_or_create_text("content")?;
let map  = doc.get_or_create_map("meta")?;
let arr  = doc.get_or_create_array("items")?;

// Edit text
text.insert(0, "Hello")?;
text.insert(5, ", world")?;

// Edit map
map.insert("title".to_string(), Any::String("My Doc".into()))?;
map.insert("created".to_string(), Any::BigInt(1234567890))?;

// Edit array
arr.insert(0, Any::String("first item".into()))?;

// Encode the current state as an update binary
let update_bytes: Vec<u8> = doc.encode_update_v1()?;

// Apply an update received from another peer
doc.apply_update_from_binary_v1(&remote_update_bytes)?;
```

---

## 4. Sync protocol

y-octo implements the standard Yjs sync protocol.

### Step 1 (client → server): send state vector

```rust
// Get a compact summary of what this doc has
let state_vector: Vec<u8> = doc.encode_state_vector_v1()?;
// Send this to the remote peer
```

### Step 2 (server → client): send missing updates

```rust
// Server receives the client's state vector and computes what they're missing
let missing_update: Vec<u8> = doc.encode_state_as_update_v1(&client_state_vector)?;
// Send missing_update back to the client
```

### Applying received updates

```rust
doc.apply_update_from_binary_v1(&missing_update)?;
```

### Merging multiple updates into one

```rust
use y_octo::merge_updates_v1;

let merged: Vec<u8> = merge_updates_v1(&[update1, update2, update3])?;
```

This is exactly what `mergeUpdatesInApplyWay` in `@affine/server-native` calls.

### Protocol messages (for WebSocket transport)

```rust
use y_octo::{SyncMessage, encode_update_as_message, encode_awareness_as_message};

// Encode for transport
let msg_bytes = encode_update_as_message(&update_bytes)?;

// Decode received message
let msg = SyncMessage::from_bytes(&received_bytes)?;
match msg {
    SyncMessage::Doc(DocMessage::Step1 { state_vector }) => { ... }
    SyncMessage::Doc(DocMessage::Step2 { update }) => { ... }
    SyncMessage::Doc(DocMessage::Update { update }) => { ... }
    SyncMessage::Awareness(awareness_update) => { ... }
}
```

---

## 5. Awareness

Awareness tracks real-time presence (cursors, selections, online status). Unlike document state, it is not persisted — it exists only while a user is connected.

```rust
use y_octo::{Awareness, AwarenessState};

let awareness = Awareness::new(doc.client_id());

// Update local state
awareness.set_local_state(AwarenessState::new(
    serde_json::json!({ "cursor": { "index": 42 } })
))?;

// Encode for broadcast
let update: Vec<u8> = awareness.encode_update()?;

// Apply received awareness update
awareness.apply_update(&remote_update)?;

// Subscribe to changes
awareness.on(|event: AwarenessEvent| {
    println!("awareness changed: {:?}", event);
})?;
```

---

## 6. History / undo

```rust
use y_octo::{History, HistoryOptions};

let history = doc.create_history(HistoryOptions {
    capture_timeout_ms: 500,  // merge edits within 500ms into one undo entry
    ..Default::default()
})?;

// Undo / redo
history.undo()?;
history.redo()?;

// Check what's available
let can_undo = history.can_undo();
let can_redo = history.can_redo();
```

---

## 7. Wire compatibility with Yjs JS

y-octo is designed to be fully wire-compatible. You can:

- Apply a `doc.encode_update_v1()` from y-octo using `Y.applyUpdate()` in Yjs JS.
- Apply a Yjs JS `Y.encodeStateAsUpdate()` using y-octo's `apply_update_from_binary_v1`.

The AFFiNE test suite (`y-octo-utils/src/compatibility_test.rs`) runs Yjs and y-octo against the same sequences of edits and asserts identical output.

---

## 8. y-octo-utils — tooling

**Package:** `packages/common/y-octo/utils/`

Utilities for development and debugging. Not for application use.

### `doc_merger` binary

Merges multiple Yjs update files on the filesystem:

```bash
cargo run --bin doc_merger -- --input updates/ --output merged.yjsupdate
```

### Benchmarks

```bash
cargo bench -p y-octo
```

Benchmarks cover: `array_ops`, `map_ops`, `text_ops`, `codec` (encode/decode), `apply` (update application), `update` (state vector + diff).

### Memory leak test

```bash
cargo run --bin memory_leak_test
```

Runs a long loop of doc operations and reports memory growth — used to catch GC issues.
