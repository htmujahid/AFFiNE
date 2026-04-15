# @affine/s3-compat — Code Documentation

S3-compatible object storage client used by the AFFiNE server to store and retrieve blobs (user-uploaded images, attachments, workspace avatars). Supports any S3-compatible service: AWS S3, Cloudflare R2, MinIO, etc.

---

## Table of Contents

1. [Creating a client](#1-creating-a-client)
2. [Basic operations](#2-basic-operations)
3. [Multipart upload](#3-multipart-upload)
4. [Presigned URLs](#4-presigned-urls)
5. [Configuration reference](#5-configuration-reference)

---

## 1. Creating a client

```ts
import { S3Compat } from '@affine/s3-compat';

const client = new S3Compat(
  {
    endpoint: 'https://s3.us-east-1.amazonaws.com',
    region: 'us-east-1',
    bucket: 'my-affine-blobs',
    forcePathStyle: false, // virtual-hosted style (default)
    requestTimeoutMs: 30_000,
  },
  {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
    sessionToken: process.env.AWS_SESSION_TOKEN, // optional, for STS
  }
);
```

For Cloudflare R2 (path-style required):

```ts
const r2 = new S3Compat(
  {
    endpoint: 'https://<account-id>.r2.cloudflarestorage.com',
    region: 'auto',
    bucket: 'affine-blobs',
    forcePathStyle: true,
  },
  { accessKeyId: R2_KEY, secretAccessKey: R2_SECRET }
);
```

---

## 2. Basic operations

### Upload

```ts
await client.putObject({
  key: 'workspaces/abc/blobs/sha256-hash',
  body: fileBuffer, // Buffer | Uint8Array | ReadableStream
  contentType: 'image/png',
  contentLength: fileBuffer.length,
});
```

### Download

```ts
const response = await client.getObjectResponse('workspaces/abc/blobs/sha256-hash');

if (response) {
  const arrayBuffer = await response.arrayBuffer();
  const contentType = response.headers.get('content-type');
}
// Returns null if the object doesn't exist
```

### Check existence / metadata

```ts
const meta = await client.headObject('workspaces/abc/blobs/sha256-hash');
// meta.contentLength, meta.contentType, meta.lastModified
// Returns null if not found
```

### Delete

```ts
await client.deleteObject('workspaces/abc/blobs/sha256-hash');
```

### List objects

```ts
const objects = await client.listObjectsV2({
  prefix: 'workspaces/abc/blobs/',
  maxKeys: 1000,
  continuationToken: '...', // for pagination
});

for (const obj of objects.contents) {
  console.log(obj.key, obj.contentLength);
}
console.log(objects.isTruncated, objects.nextContinuationToken);
```

---

## 3. Multipart upload

For large files (>5 MB recommended, required >5 GB). S3 requires at least two parts for multipart — each part except the last must be ≥5 MB.

```ts
// 1. Start
const { uploadId } = await client.createMultipartUpload({
  key: 'large-file.bin',
  contentType: 'application/octet-stream',
});

// 2. Upload parts (can be done in parallel)
const parts: { partNumber: number; etag: string }[] = [];

for (let i = 0; i < chunks.length; i++) {
  const { etag } = await client.uploadPart({
    key: 'large-file.bin',
    uploadId,
    partNumber: i + 1, // 1-indexed
    body: chunks[i],
    contentLength: chunks[i].length,
  });
  parts.push({ partNumber: i + 1, etag });
}

// 3. Complete
await client.completeMultipartUpload({
  key: 'large-file.bin',
  uploadId,
  parts,
});

// On failure — abort to avoid storage charges
await client.abortMultipartUpload({ key: 'large-file.bin', uploadId });
```

---

## 4. Presigned URLs

Generate time-limited URLs for direct client-to-S3 upload or download, bypassing your server.

```ts
// Presigned download URL (GET)
const { url, expiresAt } = await client.presignGetObject({
  key: 'workspaces/abc/blobs/sha256-hash',
  expiresIn: 3600, // seconds
});
// Give this URL to the frontend — they fetch it directly from S3

// Presigned upload URL (PUT)
const { url, headers } = await client.presignPutObject({
  key: 'workspaces/abc/blobs/new-blob',
  contentType: 'image/jpeg',
  contentLength: 204800,
  expiresIn: 300,
});
// Frontend does: fetch(url, { method: 'PUT', headers, body: file })

// Presigned multipart upload part
const { url } = await client.presignUploadPart({
  key: 'large-file.bin',
  uploadId,
  partNumber: 1,
  expiresIn: 3600,
});
```

---

## 5. Configuration reference

```ts
interface S3CompatConfig {
  endpoint: string; // full URL to S3 endpoint
  region: string; // e.g. 'us-east-1' or 'auto' for R2
  bucket: string;
  forcePathStyle?: boolean; // true for MinIO, R2 (path: /bucket/key)
  // false for AWS S3 (host: bucket.s3.region.amazonaws.com/key)
  requestTimeoutMs?: number; // default: no timeout
  minPartSize?: number; // minimum multipart part size in bytes (default: 5 MB)
  presign?: {
    expiresIn?: number; // default presign expiry in seconds
  };
}

interface S3CompatCredentials {
  accessKeyId: string;
  secretAccessKey: string;
  sessionToken?: string; // for temporary credentials (AWS STS, IAM roles)
}
```

### Signing

All requests are signed with AWS Signature Version 4 (`aws4` library). The signature covers:

- Request method + URL
- Selected headers (host, content-type, content-length, x-amz-\*)
- Payload hash (or `UNSIGNED-PAYLOAD` for streaming uploads)
