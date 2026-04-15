# affine_common — Code Documentation

A pure Rust library providing document loading, Yjs document parsing, code syntax analysis, and hashcash proof-of-work. Used by both the server-side NAPI addon (`@affine/server-native`) and WASM builds.

---

## Table of Contents

1. [Feature gates](#1-feature-gates)
2. [doc_parser — Yjs document parsing](#2-doc_parser--yjs-document-parsing)
3. [doc_loader — multi-format file loading](#3-doc_loader--multi-format-file-loading)
4. [hashcash — proof-of-work](#4-hashcash--proof-of-work)
5. [napi_utils — NAPI helpers](#5-napi_utils--napi-helpers)

---

## 1. Feature gates

The crate uses Cargo feature flags to compile only what's needed:

| Feature       | What it enables                                            |
| ------------- | ---------------------------------------------------------- |
| `doc-loader`  | `doc_loader` module — DOCX, PDF, HTML, source code loaders |
| `ydoc-loader` | `doc_parser` module — Yjs doc binary → block tree          |
| `hashcash`    | Proof-of-work challenge/response                           |
| `napi`        | Node.js NAPI binding utilities                             |

Consumers select features in `Cargo.toml`:

```toml
affine_common = { workspace = true, features = ["doc-loader", "ydoc-loader"] }
```

---

## 2. doc_parser — Yjs document parsing

**Feature:** `ydoc-loader`

Decodes a Yjs document binary and extracts AFFiNE block structure from it. This is the Rust equivalent of `@affine/reader` (the TypeScript package) — same output, faster execution.

### Functions

```rust
// Parse a binary → flat list of blocks
pub fn parse_doc_from_binary(binary: &[u8]) -> Result<CrawlResult, ParseError>

// Parse → markdown string
pub fn parse_doc_to_markdown(binary: &[u8]) -> Result<MarkdownResult, ParseError>

// Parse a page doc specifically
pub fn parse_page_doc(doc: &Doc) -> Result<PageDocContent, ParseError>

// Parse the workspace root doc
pub fn parse_workspace_doc(doc: &Doc) -> Result<WorkspaceDocContent, ParseError>

// Get all doc IDs referenced in a workspace root doc binary
pub fn get_doc_ids_from_binary(binary: &[u8]) -> Result<Vec<String>, ParseError>
```

### Write operations

For generating or modifying Yjs documents from Rust:

```rust
pub fn build_full_doc(content: &DocContent) -> Result<Vec<u8>, BuildError>
pub fn add_doc_to_root_doc(root_binary: &[u8], doc_id: &str) -> Result<Vec<u8>, BuildError>
pub fn update_doc(binary: &[u8], update: &DocUpdate) -> Result<Vec<u8>, BuildError>
pub fn update_doc_title(binary: &[u8], title: &str) -> Result<Vec<u8>, BuildError>
pub fn update_root_doc_meta_title(root_binary: &[u8], doc_id: &str, title: &str) -> Result<Vec<u8>, BuildError>
```

### Output types

```rust
pub struct BlockInfo {
    pub block_id: String,
    pub flavour: String,
    pub content: Vec<String>,         // extracted text
    pub blob: Vec<String>,            // referenced blob keys
    pub ref_doc_id: Vec<String>,      // linked doc IDs
    pub ref_info: Vec<String>,        // external URLs
    pub parent_flavour: Option<String>,
    pub parent_block_id: Option<String>,
    pub additional: Option<String>,
}

pub struct CrawlResult {
    pub blocks: Vec<BlockInfo>,
    pub title: String,
    pub summary: String,
}

pub struct MarkdownResult {
    pub markdown: String,
    pub title: String,
}
```

### When to use this vs `@affine/reader`

- **Server-side indexer** → use this (Rust, faster, no JS runtime needed).
- **Browser-side search** → use `@affine/reader` (TypeScript, runs in the browser).
- Both produce equivalent output.

---

## 3. doc_loader — multi-format file loading

**Feature:** `doc-loader`

Loads and extracts text from various file formats. Used by the AI copilot to process user-uploaded documents as context.

### Supported formats

| Format          | Loader                                            |
| --------------- | ------------------------------------------------- |
| `.docx`         | `DocxLoader` — parses Word XML                    |
| `.pdf`          | `PdfExtractLoader` — text layer extraction        |
| `.html`, `.htm` | `HtmlLoader` — strips tags, extracts text         |
| `.txt`, `.md`   | `TextLoader` — plain text                         |
| Source code     | `SourceCodeLoader` — syntax-aware via tree-sitter |

### Source code language detection

```rust
// Returns the tree-sitter language for a filename
pub fn get_language_by_filename(filename: &str) -> Option<Language>
```

Supported languages: C, C#, C++, Go, Java, JavaScript, Kotlin, Python, Rust, Scala, TypeScript.

### Output types

```rust
pub struct Chunk {
    pub text: String,
    pub metadata: HashMap<String, String>,  // page number, section, etc.
}

pub struct Doc {
    pub chunks: Vec<Chunk>,
    pub mime: String,
    pub filename: String,
}

pub enum LoaderError {
    UnsupportedFormat,
    ParseError(String),
    IoError(std::io::Error),
}

pub type LoaderResult = Result<Doc, LoaderError>;
```

### Usage pattern (from server side)

```rust
use affine_common::doc_loader::{DocxLoader, PdfExtractLoader, HtmlLoader, Loader};

let bytes = std::fs::read("document.docx")?;
let doc = DocxLoader.load(&bytes, "document.docx")?;

for chunk in &doc.chunks {
    println!("{}", chunk.text);
}
```

---

## 4. hashcash — proof-of-work

**Feature:** `hashcash`

Implements a SHA3-based hashcash scheme for anti-abuse challenges (sign-up, sensitive API endpoints).

### How it works

1. Server issues a challenge string.
2. Client computes a nonce such that `SHA3(challenge + nonce)` has N leading zero bits.
3. Server verifies the response — cheap to verify, expensive to compute.

```rust
// Server: verify a response
pub fn verify_challenge_response(
    challenge: &str,
    response: &str,
    difficulty: u8,          // number of required leading zero bits
) -> bool

// Client (Rust side, also available in TypeScript via NAPI):
pub fn mint_challenge_response(
    challenge: &str,
    difficulty: u8,
) -> String
```

The difficulty is typically 16–20 bits (takes ~1 second on modern hardware).

---

## 5. napi_utils — NAPI helpers

**Feature:** `napi`

Utility functions for the NAPI addon crates (`affine_server_native`, `affine_mobile_native`):

```rust
// Map Rust errors to NAPI errors with a given status code
pub fn map_napi_err<T>(result: Result<T, E>, status: napi::Status) -> napi::Result<T>
```

This is infrastructure code — you only need it when writing new NAPI bindings.
