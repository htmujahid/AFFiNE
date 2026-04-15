# @affine/reader — Code Documentation

Reads AFFiNE / BlockSuite Yjs documents and extracts structured block content: text, references, blob keys, markdown previews. Used by the search indexer and content analysis pipelines.

---

## Table of Contents

1. [What it does](#1-what-it-does)
2. [Main functions](#2-main-functions)
3. [BlockDocumentInfo — the output type](#3-blockdocumentinfo--the-output-type)
4. [Block flavours handled](#4-block-flavours-handled)
5. [Markdown preview generation](#5-markdown-preview-generation)

---

## 1. What it does

A BlockSuite document is stored as a Yjs doc — a nested `YMap` tree of blocks. This package traverses that tree and produces plain JavaScript objects you can index, search, or render.

It does **not** render UI. It is pure extraction logic: Yjs in, structured data out.

---

## 2. Main functions

```ts
import { readAllDocsFromRootDoc, readAllBlocksFromDoc } from '@affine/reader';
import * as Y from 'yjs';

// If you have a root workspace doc (contains references to all page docs)
const rootDoc = new Y.Doc();
Y.applyUpdate(rootDoc, workspaceSnapshotBinary);

const blockInfos: BlockDocumentInfo[] = readAllDocsFromRootDoc(rootDoc);
// Returns blocks from ALL docs in the workspace

// If you already have a single page doc
const pageDoc = new Y.Doc();
Y.applyUpdate(pageDoc, pageSnapshotBinary);

const blocks: BlockDocumentInfo[] = readAllBlocksFromDoc('page-doc-id', pageDoc);
```

---

## 3. BlockDocumentInfo — the output type

```ts
interface BlockDocumentInfo {
  docId: string; // which doc this block belongs to
  blockId: string; // unique block ID within the doc
  flavour: string; // block type (see section 4)
  content?: string[]; // extracted text content (paragraphs, headings, etc.)
  blob?: string[]; // blob keys referenced by this block (image src, attachment)
  refDocId?: string[]; // linked doc IDs (internal page links)
  ref?: string[]; // external URLs (bookmarks, embeds)
  parentFlavour?: string; // parent block's flavour
  parentBlockId?: string; // parent block's ID
  yblock: Y.Map<unknown>; // raw Yjs map reference (for advanced use)
  markdownPreview?: string; // markdown representation of this block's content
}
```

---

## 4. Block flavours handled

| Flavour                   | Content extracted                    |
| ------------------------- | ------------------------------------ |
| `affine:paragraph`        | Text delta → plain string            |
| `affine:heading`          | Text delta → plain string            |
| `affine:list`             | Text delta → plain string            |
| `affine:code`             | Code content                         |
| `affine:image`            | `blob` key (the image's storage key) |
| `affine:attachment`       | `blob` key, file name                |
| `affine:bookmark`         | `ref` URL, title                     |
| `affine:embed-youtube`    | `ref` URL                            |
| `affine:embed-figma`      | `ref` URL                            |
| `affine:embed-github`     | `ref` URL                            |
| `affine:embed-loom`       | `ref` URL                            |
| `affine:embed-linked-doc` | `refDocId` (internal link)           |
| `affine:database`         | Column headers + row text            |
| `affine:note`             | Container — children extracted       |
| `affine:page`             | Root container                       |

Inline references inside paragraph text (e.g., `[[Page Title]]` mention links) are also extracted as `refDocId` entries by scanning Yjs text delta attributes.

---

## 5. Markdown preview generation

When `markdownPreview` is populated, it uses BlockSuite's `MarkdownAdapter` to convert the block's Yjs content to a Markdown string. This is used by:

- Search result snippets
- AI copilot context injection
- Doc summary generation

The conversion is done via BlockSuite's schema and transformer system, which is initialised by registering `AffineSchemas` before calling the reader functions.

---

## Typical use case: indexing pipeline

```ts
// In the server indexer or Electron search worker
import { readAllBlocksFromDoc } from '@affine/reader';
import * as Y from 'yjs';

async function indexDoc(docId: string, snapshotBin: Uint8Array) {
  const doc = new Y.Doc();
  Y.applyUpdate(doc, snapshotBin);

  const blocks = readAllBlocksFromDoc(docId, doc);

  const indexEntries = blocks
    .filter(b => b.content?.length)
    .map(b => ({
      docId: b.docId,
      blockId: b.blockId,
      text: b.content!.join(' '),
      links: b.refDocId ?? [],
      blobs: b.blob ?? [],
    }));

  await searchIndex.upsert(docId, indexEntries);
}
```
