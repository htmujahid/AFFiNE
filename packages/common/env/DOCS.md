# @affine/env — Code Documentation

Centralized environment constants, build-time flags, platform detection, and shared type definitions used across all AFFiNE packages.

---

## Table of Contents

1. [BUILD_CONFIG — build-time flags](#1-build_config--build-time-flags)
2. [setupGlobal — platform detection](#2-setupglobal--platform-detection)
3. [Constants](#3-constants)
4. [MessageCode — user-facing error codes](#4-messagecode--user-facing-error-codes)
5. [Shared types](#5-shared-types)

---

## 1. BUILD_CONFIG — build-time flags

`BUILD_CONFIG` is a global object injected at build time by the Rspack/Vite config. It contains values that are **constant for a given build** — they cannot change at runtime.

```ts
declare const BUILD_CONFIG: {
  debug: boolean; // true in development builds
  isElectron: boolean; // true when built for Electron
  isMobileWeb: boolean; // true when built for mobile web
  isIOS: boolean; // true when built for iOS (Capacitor)
  isAndroid: boolean; // true when built for Android (Capacitor)
  appVersion: string; // semver, e.g. '0.26.3'
  editorVersion: string; // BlockSuite version
  appBuildType: 'stable' | 'beta' | 'internal' | 'canary';
  serverUrlPrefix: string; // e.g. 'https://app.affine.pro'
  githubUrl: string;
  changelogUrl: string;
  downloadUrl: string;
  // ... other static URLs
};
```

These are replaced with literal values at build time, so dead-code elimination removes branches that can never be true:

```ts
// In Electron build, the entire block is removed from the bundle
if (!BUILD_CONFIG.isElectron) {
  // ... browser-only code
}
```

---

## 2. `setupGlobal` — platform detection

**File:** `src/global.ts`

Call once at app startup to populate the `$AFFINE_SETUP` global:

```ts
import { setupGlobal } from '@affine/env/global';

setupGlobal(); // idempotent — safe to call multiple times
```

After calling, read platform info from the global:

```ts
const setup = globalThis.$AFFINE_SETUP;

setup.isLinux; // → boolean
setup.isMacOs; // → boolean
setup.isWindows; // → boolean
setup.isSafari; // → boolean
setup.isFirefox; // → boolean
setup.isChrome; // → boolean
setup.isMobile; // → boolean (phone/tablet)
setup.isPwa; // → boolean (installed as PWA)
setup.isSelfHosted; // → boolean (running on a custom domain)
setup.chromeVersion; // → number | null
```

Detection is based on User-Agent parsing. The `isElectron`/`isIOS`/`isAndroid` values come from `BUILD_CONFIG` (compile-time), not UA detection.

---

## 3. Constants

**File:** `src/constant.ts`

```ts
// Default names used when creating new workspaces/docs
DEFAULT_WORKSPACE_NAME = 'My AFFiNE';
UNTITLED_WORKSPACE_NAME = 'Untitled Workspace';
UNTITLED_PAGE_NAME = 'Untitled';

// Default sort key for doc lists
DEFAULT_SORT_KEY = 'updatedDate';
```

---

## 4. MessageCode — user-facing error codes

Enum of symbolic codes used to look up user-facing error messages:

```ts
enum MessageCode {
  loginError,
  noPermission,
  loadListFailed,
  blobTooLarge,
  // ...
}

const Messages: Record<MessageCode, string> = {
  [MessageCode.blobTooLarge]: 'File size exceeds the limit.',
  // ...
};
```

Use `Messages[MessageCode.blobTooLarge]` to get the display string. For GraphQL-originated errors, prefer `@affine/error`'s `UserFriendlyError` which carries the message from the server.

---

## 5. Shared types

**`src/workspace.ts`** — workspace-level type constants (workspace roles, member statuses).

**`src/filter.ts`** — filter/sort types used by doc list views.

**`src/automation.ts`** — `Action<InputSchema, Args>` type for the automation system. Uses Zod for runtime schema validation:

```ts
import type { Action } from '@affine/env/automation';

const myAction: Action<typeof MySchema, MyArgs> = {
  schema: MySchema,    // Zod schema for input validation
  handler: async (input) => { ... },
};
```

**`src/blocksuite/`** — types bridging AFFiNE core with BlockSuite (block flavours, slot types, etc.).
