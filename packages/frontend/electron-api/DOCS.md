# @affine/electron-api — Code Documentation

Typed IPC bridge between the Electron renderer process and the main/helper processes. Provides `window.apis` (callable handlers) and `window.events` (subscribable events) with full TypeScript types.

---

## Table of Contents

1. [How IPC works in AFFiNE Electron](#1-how-ipc-works-in-affine-electron)
2. [Calling main process handlers](#2-calling-main-process-handlers)
3. [Subscribing to main process events](#3-subscribing-to-main-process-events)
4. [AppInfo — build metadata](#4-appinfo--build-metadata)
5. [SharedStorage — cross-process storage](#5-sharedstorage--cross-process-storage)
6. [Type reference](#6-type-reference)

---

## 1. How IPC works in AFFiNE Electron

```
Renderer (this package)        Preload script             Main process
    window.apis.*         →   contextBridge.exposeInMainWorld  →  ipcMain.handle()
    window.events.*       ←   contextBridge.exposeInMainWorld  ←  ipcMain.emit()
```

The preload script (in `@affine/electron/preload/electron-api.ts`) exposes the bridge. This package provides the TypeScript types for what's on the other side.

In application code, the bridge is accessed through `DesktopApiService` from `@affine/core`:

```ts
const desktopApi = useService(DesktopApiService);
await desktopApi.handler.ui.openFilePicker();
```

---

## 2. Calling main process handlers

`window.apis` / `desktopApi.handler` is an object of async functions. Categories:

### `ui.*` — Window and UI operations

```ts
await desktopApi.handler.ui.openFilePicker(); // → string[] (selected file paths)
await desktopApi.handler.ui.revealInFinder(path);
await desktopApi.handler.ui.moveToTrash(path);
await desktopApi.handler.ui.showDevTools();
await desktopApi.handler.ui.setWindowSize(width, height);
await desktopApi.handler.ui.getWindowSize(); // → { width, height }
```

### `app.*` — Application lifecycle

```ts
await desktopApi.handler.app.quit();
await desktopApi.handler.app.getAppInfo(); // → AppInfo
await desktopApi.handler.app.relaunch();
```

### `updater.*` — Auto-updater

```ts
await desktopApi.handler.updater.checkForUpdates(); // → UpdateMeta | null
await desktopApi.handler.updater.downloadUpdate();
await desktopApi.handler.updater.quitAndInstall();
```

### `dialog.*` — Native dialogs

```ts
await desktopApi.handler.dialog.showOpenDialog(options); // Electron.OpenDialogOptions
await desktopApi.handler.dialog.showSaveDialog(options);
await desktopApi.handler.dialog.showMessageBox(options);
```

### `tabs.*` — Tab management

```ts
await desktopApi.handler.tabs.openTab(option: AddTabOption)
await desktopApi.handler.tabs.closeTab(key: string)
await desktopApi.handler.tabs.activateTab(key: string)
await desktopApi.handler.tabs.getTabsInfo()             // → TabViewsMetaSchema
```

### `nbstore.*` — Document database (helper process)

```ts
await desktopApi.handler.nbstore.connect(universalId, dbPath);
await desktopApi.handler.nbstore.pushDocUpdate(universalId, docId, update);
await desktopApi.handler.nbstore.getDocSnapshot(universalId, docId);
await desktopApi.handler.nbstore.getBlob(universalId, key);
await desktopApi.handler.nbstore.setBlob(universalId, blob);
```

---

## 3. Subscribing to main process events

`window.events` / `desktopApi.events` are event emitters. Subscribe:

```ts
// Returns unsubscribe function
const unsub = desktopApi.events.updater.onUpdateAvailable(meta => {
  showUpdateBanner(meta.version);
});

// On component unmount
unsub();
```

Key events:

| Namespace | Event                | Payload              |
| --------- | -------------------- | -------------------- |
| `updater` | `onUpdateAvailable`  | `UpdateMeta`         |
| `updater` | `onUpdateDownloaded` | `UpdateMeta`         |
| `app`     | `onAppActivated`     | —                    |
| `app`     | `onDeepLink`         | `string` (the URL)   |
| `tabs`    | `onTabsChanged`      | `TabViewsMetaSchema` |
| `tabs`    | `onActiveTabChanged` | `string` (tab key)   |
| `ui`      | `onWindowResize`     | `{ width, height }`  |

---

## 4. AppInfo — build metadata

```ts
interface AppInfo {
  appName: string; // 'AFFiNE'
  buildType: 'stable' | 'beta' | 'internal' | 'canary';
  version: string; // semver, e.g. '0.26.3'
  isMAS: boolean; // Mac App Store build
  platform: 'darwin' | 'win32' | 'linux';
  arch: string; // 'x64' | 'arm64'
}

// Access via:
const info = window.__appInfo;
// or
const info = await desktopApi.handler.app.getAppInfo();
```

---

## 5. SharedStorage — cross-process storage

A persistent key-value store that is synchronised across the main and renderer processes:

```ts
interface SharedStorage {
  get(key: string): unknown;
  set(key: string, value: unknown): void;
  delete(key: string): void;
  onChange(key: string, callback: (value: unknown) => void): () => void;
}

// Access via:
window.__sharedStorage.get('lastWorkspaceId');
window.__sharedStorage.set('theme', 'dark');
```

Used for settings that need to survive renderer restarts without a round-trip to main.

---

## 6. Type reference

Core types re-exported from `@affine/electron`:

```ts
// Window tab schema
type TabViewsMetaSchema = {
  tabs: WorkbenchMeta[];
  activeTabKey: string;
};

type WorkbenchMeta = {
  key: string;
  views: WorkbenchViewMeta[];
  activeViewIndex: number;
  pinned?: boolean;
};

type WorkbenchViewMeta = {
  id: string;
  title?: string;
  path: { pathname: string; search?: string; hash?: string };
  module?: WorkbenchViewModule;
};

// Auto-updater
type UpdateMeta = {
  version: string;
  releaseDate: string;
  releaseNotes: string;
};

// Tab management
type AddTabOption = {
  view?: { path: string; title?: string };
  edge?: 'left' | 'right';
  show?: boolean;
  pinned?: boolean;
  basename?: string;
};
```
