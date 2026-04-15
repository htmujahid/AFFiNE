# @affine/mobile-shared — Code Documentation

Shared utilities between the iOS and Android native apps. Primarily handles the encoding and decoding of large binary payloads passed through the Capacitor bridge.

---

## The blob token pattern

When native code (Swift/Kotlin) has a large binary (e.g., a document snapshot or blob) that needs to be returned to JavaScript, passing it as a base64 string is slow and memory-intensive for blobs > 1 MB.

Instead, the native side writes the binary to a temp file and returns a **token string**:

```
__AFFINE_BLOB_FILE__:/var/mobile/.../tmp/blob-abc123.bin
```

JavaScript receives this token and uses `@affine/mobile-shared` to decode it:

```ts
import { decodeBlobPayload, isBlobFileToken } from '@affine/mobile-shared/nbstore/payload';

const rawValue = await nativePlugin.getBlob({ key: 'sha256-abc' });

if (isBlobFileToken(rawValue)) {
  // Large blob: read from temp file path
  const blobData: Uint8Array = await decodeBlobPayload(rawValue);
} else {
  // Small blob: rawValue is already a base64 string
  const blobData = base64ToUint8Array(rawValue);
}
```

---

## API

```ts
// Check if a string is a blob file token
function isBlobFileToken(value: string): boolean;

// Decode a token (reads the file) or decode a plain base64 string
async function decodeBlobPayload(payload: string): Promise<Uint8Array>;

// Token prefix constant
const BLOB_FILE_TOKEN_PREFIX = '__AFFINE_BLOB_FILE__:';
```

---

## When is this used?

The Capacitor plugins for `@affine/ios` and `@affine/android` use `affine_mobile_native` (Rust/UniFFI) for storage. When `storageGetBlob()` or `storageGetDocSnapshot()` returns large data, the Rust side writes to a temp file and returns a token. This package decodes it back in JavaScript before it reaches application code.

Application code (in `@affine/core`) never sees tokens — they're transparently decoded by the storage bridge layer.
