# @affine/server-native — Code Documentation

A Rust NAPI-RS addon compiled to a `.node` binary that the NestJS server loads at startup. It offloads performance-critical or CPU-heavy operations from Node.js to Rust, keeping the event loop free.

---

## Table of Contents

1. [Why a native addon?](#1-why-a-native-addon)
2. [How it's loaded](#2-how-its-loaded)
3. [Module reference](#3-module-reference)
4. [LLM dispatcher — deep dive](#4-llm-dispatcher--deep-dive)
5. [Yjs document operations](#5-yjs-document-operations)
6. [Image processing](#6-image-processing)
7. [Tokenizer](#7-tokenizer)
8. [File type detection](#8-file-type-detection)
9. [HTML sanitizer](#9-html-sanitizer)
10. [Hashcash](#10-hashcash)
11. [Doc loader](#11-doc-loader)
12. [Build and release notes](#12-build-and-release-notes)

---

## 1. Why a native addon?

Three categories of work benefit from being in Rust:

| Category         | Examples                                         | Why Rust?                                                                                         |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| CPU-bound        | Yjs update merge, image resize, tokenizer        | Avoids V8 GC pauses, true parallelism via OS threads                                              |
| Blocking I/O     | LLM HTTP streaming                               | Runs on a Rust thread, returns events to Node.js via thread-safe callbacks; event loop stays free |
| Low-level codecs | WebP encoding, libwebp, EXIF, file-type sniffing | Mature C/Rust libs, no npm equivalent with same performance                                       |

---

## 2. How it's loaded

The server loads the native addon via `src/native.ts`:

```ts
// packages/backend/server/src/native.ts
import native from '@affine/server-native';
export const { mergeUpdatesInApplyWay, processImage, fromModelName, ... } = native;
```

`@affine/server-native` resolves to the compiled `affine_server_native.node` binary. NAPI-RS generates TypeScript type declarations alongside it so all exported functions are typed.

The addon uses `mimalloc` as its global allocator for better multi-threaded allocation performance (default Linux `glibc` allocator has lock contention under load).

---

## 3. Module reference

| Rust file          | Exported API                                                                                             | Used by                                         |
| ------------------ | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `lib.rs`           | `mergeUpdatesInApplyWay()`, license constants                                                            | `core/doc` snapshot merge                       |
| `llm.rs`           | `llmDispatch`, `llmDispatchStream`, `llmEmbeddingDispatch`, `llmRerankDispatch`, `llmStructuredDispatch` | `plugins/copilot` native provider               |
| `image.rs`         | `processImage()`                                                                                         | `core/storage` blob upload pipeline             |
| `tiktoken.rs`      | `fromModelName()` → `Tokenizer.count()`                                                                  | `plugins/copilot` context window management     |
| `file_type.rs`     | `getFileType()`                                                                                          | `core/storage` MIME validation                  |
| `html_sanitize.rs` | `sanitizeHtml()`                                                                                         | `plugins/copilot` tool outputs                  |
| `hashcash.rs`      | `mintChallengeResponse()`                                                                                | Anti-abuse PoW challenges                       |
| `doc_loader.rs`    | `loadDoc()`                                                                                              | `plugins/indexer` text extraction from Yjs docs |
| `doc.rs`           | helpers for doc-level operations                                                                         | Internal, re-exported from `lib.rs`             |

---

## 4. LLM dispatcher — deep dive

The most complex module. It implements a portable HTTP client for multiple LLM provider APIs, callable from Node.js, running entirely on Rust threads.

### Supported protocols

Controlled by the `protocol` string argument:

| Protocol alias                                        | Maps to                     |
| ----------------------------------------------------- | --------------------------- |
| `openai_chat`, `chat-completions`, `chat_completions` | OpenAI Chat Completions API |
| `openai_responses`, `responses`                       | OpenAI Responses API        |
| `anthropic`, `anthropic_messages`                     | Anthropic Messages API      |
| `gemini`, `gemini_generate_content`                   | Google Gemini API           |

### Exported functions

```ts
// One-shot request → string (JSON response)
llmDispatch(protocol, backendConfigJson, requestJson): Promise<string>

// Structured output request (forces JSON schema response)
llmStructuredDispatch(protocol, backendConfigJson, requestJson): Promise<string>

// Embedding vectors
llmEmbeddingDispatch(protocol, backendConfigJson, requestJson): Promise<string>

// Reranking
llmRerankDispatch(protocol, backendConfigJson, requestJson): Promise<string>

// Streaming — calls callback with JSON-serialized StreamEvent strings
// Returns LlmStreamHandle with .abort() method
llmDispatchStream(
  protocol,
  backendConfigJson,
  requestJson,
  callback: (event: string) => void
): LlmStreamHandle
```

### Middleware pipeline

Every request passes through two pipelines:

**Request middleware** (applied before sending to provider):

| Middleware            | What it does                                                    |
| --------------------- | --------------------------------------------------------------- |
| `normalize_messages`  | Normalises message roles, merges consecutive same-role messages |
| `tool_schema_rewrite` | Rewrites tool schemas to provider-specific format               |
| `clamp_max_tokens`    | Caps `max_tokens` to provider model limits                      |

**Stream middleware** (applied to each SSE event):

| Middleware               | What it does                                                               |
| ------------------------ | -------------------------------------------------------------------------- |
| `stream_event_normalize` | Normalises provider-specific event formats to a common `StreamEvent` shape |
| `citation_indexing`      | Extracts and numbers citations from Perplexity/web-search responses        |

### Streaming event shape

The callback receives JSON strings representing a `StreamEvent` union:

```ts
// Possible event types emitted to the callback
{ type: 'text_delta', delta: string }
{ type: 'thinking_delta', delta: string }
{ type: 'tool_call', ... }
{ type: 'error', message: string, code?: string }
"__AFFINE_LLM_STREAM_END__"  // sentinel string marking stream completion
```

### Streaming implementation

```
Node.js calls llmDispatchStream(...)
  │
  └─ Rust spawns OS thread (std::thread::spawn)
       │
       ├─ applies request middlewares
       ├─ opens HTTP connection to provider
       ├─ reads SSE events from response body
       │     for each event:
       │       ├─ runs stream middleware pipeline
       │       └─ calls ThreadsafeFunction (NAPI callback) back to Node.js
       └─ emits "__AFFINE_LLM_STREAM_END__" sentinel
```

`LlmStreamHandle.abort()` sets an `AtomicBool` that the Rust thread checks between events, terminating cleanly.

### Why this over a Node.js HTTP client?

- LLM streaming holds a TCP connection open for seconds. Doing this in Node.js ties up the event loop's microtask queue during parsing.
- Multiple concurrent streams scale better with OS threads + `ureq` (synchronous Rust HTTP) than with Node.js async I/O when under high concurrency.
- The middleware logic (schema rewriting, message normalization) is shared with other Rust consumers without duplicating code in TypeScript.

---

## 5. Yjs document operations

### `mergeUpdatesInApplyWay(updates: Buffer[]): Buffer`

Located in `lib.rs`. Takes an array of raw Yjs v1 update binaries and merges them into a single snapshot binary.

**Algorithm:**

1. Creates a fresh `y-octo::Doc`
2. Applies each update via `doc.apply_update_from_binary_v1()`
3. Returns `doc.encode_update_v1()` — a single binary representing the merged state

**Why "in apply way":** This mirrors how `Y.applyUpdate(doc, update)` works in the JS Yjs library — it applies updates sequentially rather than performing a structural merge. This is simpler and guaranteed wire-compatible.

Used by `PgWorkspaceDocStorageAdapter` and `PgUserspaceDocStorageAdapter` in the snapshot merge background job.

---

## 6. Image processing

### `processImage(input: Buffer, maxEdge: number, keepExif: boolean): Promise<Buffer>`

Located in `image.rs`. Converts any supported image format to WebP, resizes if needed, and optionally strips EXIF metadata.

**Processing steps:**

1. Detect format (`image::guess_format`)
2. Check dimensions — rejects images over 16,384px or 40 megapixels
3. Auto-orient based on EXIF orientation tag
4. Resize: if `max(width, height) > maxEdge`, scales down preserving aspect ratio using Lanczos3 filter
5. Encode to WebP at 80% quality using `libwebp`
6. If `keepExif=true`, re-attaches EXIF chunk to the WebP container via `libwebp_sys::WebPMuxSetChunk`

**Special handling:**

- Animated GIF/WebP: decodes only the first frame (animation not preserved)
- EXIF orientation: applied before resize so the final image is correctly oriented regardless of input

This runs as a NAPI async task (`AsyncTask`), meaning it executes on a libuv thread pool thread and doesn't block Node.js.

---

## 7. Tokenizer

### `fromModelName(modelName: string): Tokenizer | null`

### `Tokenizer.count(content: string, allowedSpecial?: string[]): number`

Located in `tiktoken.rs`. Wraps `tiktoken-rs` (Rust port of OpenAI's tiktoken).

Returns a `Tokenizer` instance for the given model name. Returns `null` if the model is unknown.

```ts
const tokenizer = fromModelName('gpt-4o');
const count = tokenizer.count('Hello, world!'); // → 4
```

**Special case:** Model names starting with `gpt-5` are mapped to the `o200k_base` encoding (same as GPT-4o) since tiktoken-rs doesn't yet have a specific gpt-5 entry.

**Used by:** `CopilotSessionService` to count tokens in the conversation history before sending to a provider, ensuring requests stay within the model's context window.

---

## 8. File type detection

### `getFileType(buffer: Buffer): FileTypeResult`

Located in `file_type.rs`. Detects the MIME type of a binary buffer by reading magic bytes, using the `infer` and `file-format` crates.

Returns `{ mime: string, ext: string }`.

Used by the blob upload pipeline in `core/storage` to validate and normalize the content type of uploaded files before storing them.

---

## 9. HTML sanitizer

### `sanitizeHtml(html: string): string`

Located in `html_sanitize.rs`. Strips disallowed tags and attributes using `v_htmlescape` and `htmlrewriter`. Used to sanitize HTML returned by AI tool calls before it's stored or sent to clients.

---

## 10. Hashcash

### `mintChallengeResponse(...): string`

Located in `hashcash.rs`. Implements a server-side hashcash proof-of-work verifier. Used for anti-abuse challenges on sensitive endpoints (e.g., sign-up).

The client receives a challenge and must compute a response that satisfies a difficulty requirement (number of leading zero bits in `SHA3(challenge + nonce)`). The server verifies the response using this native function.

---

## 11. Doc loader

### `loadDoc(binary: Buffer): DocContent`

Located in `doc_loader.rs`. Decodes a Yjs doc binary and extracts structured text content (block tree) from it using `affine_common::doc_loader`.

**Used by:** `plugins/indexer` to extract plain text from doc snapshots for full-text indexing and embedding generation. Running this in Rust avoids spawning a JS Yjs runtime for each doc.

---

## 12. Build and release notes

The crate compiles to a `cdylib` (shared library). NAPI-RS generates:

- `affine_server_native.node` — the binary loaded by Node.js
- TypeScript `.d.ts` type declarations for all `#[napi]` exports

**Platform targets** (from `Cargo.toml`):

- `native` (dev machine)
- `debian-openssl-3.0.x` (cloud/Docker)
- `linux-arm64-openssl-3.0.x` (ARM servers / Apple Silicon in Docker)

**License constants:** `AFFINE_PRO_PUBLIC_KEY` and `AFFINE_PRO_LICENSE_AES_KEY` are baked in at compile time via `std::option_env!()`. These are only present in official AFFiNE Pro cloud builds; self-hosted builds will have `None`.

**Local dev:** The `.node` binary is pre-built and committed/downloaded. You do not need to run `cargo build` unless modifying native code. Run `yarn build --package @affine/server-native` to rebuild.
