# @affine/electron — Code Documentation

The Electron main process. Handles window creation, IPC, native menus, auto-update, deep linking, and the helper worker process that runs SQLite.

---

## Table of Contents

1. [Process architecture](#1-process-architecture)
2. [Main process modules](#2-main-process-modules)
3. [Helper process](#3-helper-process)
4. [Preload scripts](#4-preload-scripts)
5. [Startup sequence](#5-startup-sequence)
6. [Tab management](#6-tab-management)
7. [Auto-updater](#7-auto-updater)
8. [Deep linking](#8-deep-linking)

---

## 1. Process architecture

```
┌─────────────────────────────────────┐
│           Main Process               │
│  Window management, app menu,        │
│  auto-update, tray, deep links       │
│  src/main/                           │
└──────────────┬──────────────────────┘
               │ IPC (ipcMain.handle)
               │ contextBridge (preload)
               ↕
┌─────────────────────────────────────┐
│         Renderer Process             │
│  @affine/electron-renderer           │
│  runs the full @affine/core UI       │
└──────────────┬──────────────────────┘
               │ IPC (ipcRenderer.invoke)
               ↕
┌─────────────────────────────────────┐
│          Helper Process              │
│  Heavy I/O: SQLite, thumbnails,      │
│  workspace operations                │
│  src/helper/                         │
└─────────────────────────────────────┘
```

The main and helper processes never touch the renderer's DOM. All communication is typed IPC via `@affine/electron-api`.

---

## 2. Main process modules

```
src/main/
  app.ts                  ← Entry: registers everything, launches first window
  handlers.ts             ← All ipcMain.handle() definitions
  events.ts               ← ipcMain.on() event listeners
  windows-manager/
    launcher.ts           ← First window launch logic
    main-window.ts        ← BrowserWindow config and lifecycle
    tab-views.ts          ← Multi-tab management (BrowserView per tab)
    popup.ts              ← Popup window spawning (OAuth, modals)
    onboarding.ts         ← First-run experience window
    authentication.ts     ← Auth window
  application-menu/
    create.ts             ← macOS/Windows native app menu
  updater/
    index.ts              ← electron-updater integration
  config-storage/
    persist.ts            ← Persistent desktop config (electron-store)
  deep-link.ts            ← affine:// URL scheme handling
  protocol.ts             ← assets:// custom protocol (serves local files)
  clipboard/              ← Clipboard read/write operations
  find-in-page/           ← Ctrl+F find-in-page feature
  recording/
    feature.ts            ← Screen/audio recording (macOS)
  tray.ts                 ← System tray icon + context menu
  security-restrictions.ts ← Sandbox + CSP enforcement
  cleanup.ts              ← On-exit resource cleanup
  logger.ts               ← Winston logger for main process
```

---

## 3. Helper process

A separate Node.js process (`src/helper/`) for CPU/IO-heavy operations that would block the main process:

```
src/helper/
  index.ts                ← Entry: sets up RPC listeners
  nbstore/                ← SQLite document storage via @affine/native
  preview/                ← Workspace/doc thumbnail generation
  workspace/              ← Workspace migration utilities
  dialog/                 ← Native file picker dialogs
```

The helper talks to the main process via `BrowserWindow.webContents.send` / `ipcRenderer.invoke` — the same IPC transport but with a dedicated channel.

**Why a separate process?**

- SQLite reads/writes don't block UI
- Thumbnail generation doesn't freeze the window
- The process can be restarted independently if it crashes

---

## 4. Preload scripts

Two preload scripts bridge the sandboxed renderer to the outside world:

### `preload/electron-api.ts`

Exposes `window.__apis` and `window.__events` using `contextBridge.exposeInMainWorld`. These become `DesktopApiService.handler` and `DesktopApiService.events` in the renderer.

```ts
// What the renderer gets
window.__apis = {
  ui: { openFilePicker, revealInFinder, ... },
  app: { quit, getAppInfo, relaunch },
  tabs: { openTab, closeTab, activateTab },
  nbstore: { connect, pushDocUpdate, getDocSnapshot, ... },
  updater: { checkForUpdates, downloadUpdate, quitAndInstall },
  dialog: { showOpenDialog, showSaveDialog },
  // ...
}
```

### `preload/shared-storage.ts`

Exposes `window.__sharedStorage` — a key-value store synced between main and renderer.

---

## 5. Startup sequence

```
app.ts
  1. Enable sandbox (app.enableSandbox())
  2. Patch network resolution (0.0.0.0 → 127.0.0.1 for dev server)
  3. Disable autofill for security
  4. Single instance lock (app.requestSingleInstanceLock())
     If another instance → activate existing window → exit
  5. Register affine:// deep link scheme
  6. Register security restrictions (block navigation, restrict new windows)
  7. app.whenReady():
     a. Register assets:// protocol
     b. Register IPC handlers (handlers.ts)
     c. Register event listeners (events.ts)
     d. Launch first window (windows-manager/launcher.ts)
     e. Create app menu (application-menu/create.ts)
     f. Start auto-updater
     g. Initialize recording feature (macOS only)
     h. Create tray icon
```

---

## 6. Tab management

AFFiNE Electron supports multiple browser-tab-style views within a single window using Electron's `BrowserView` API.

**File:** `src/main/windows-manager/tab-views.ts`

```ts
// Open a new tab
tabsManager.openTab({
  view: { path: '/workspace/abc/page/xyz', title: 'My Doc' },
  show: true,
});

// Close a tab
tabsManager.closeTab(tabKey);

// Switch active tab
tabsManager.activateTab(tabKey);

// Pin a tab
tabsManager.pinTab(tabKey);

// Split view (two docs side by side)
tabsManager.splitView(tabKey);
```

Each `BrowserView` loads the same renderer HTML but with different URL routing. The `TabViewsMetaSchema` (in `@affine/electron-api`) is broadcast to all renderers when tabs change.

---

## 7. Auto-updater

Uses `electron-updater` with platform-specific installers:

- macOS: `dmg` / Mac App Store
- Windows: `NSIS` installer / `Squirrel`
- Linux: `AppImage` / `deb`

```ts
// Main process checks for updates on launch (production only)
autoUpdater.checkForUpdates();

// Events broadcast to renderer:
autoUpdater.on('update-available', meta => events.updater.onUpdateAvailable(meta));
autoUpdater.on('update-downloaded', meta => events.updater.onUpdateDownloaded(meta));

// Renderer triggers install:
ipcMain.handle('updater:quitAndInstall', () => {
  autoUpdater.quitAndInstall();
});
```

---

## 8. Deep linking

AFFiNE registers the `affine://` URL scheme. When a user clicks an `affine://` link:

```
affine://workspace/abc/page/xyz
  → activate app window
  → navigate renderer to the path
  → route to the specified workspace + document
```

**File:** `src/main/deep-link.ts`

On macOS, this also handles `open-url` events for clicks from outside the app.

---

## Key configuration

```ts
// src/main/config.ts
{
  dev: boolean,           // NODE_ENV === 'development'
  buildType: 'stable' | 'beta' | 'internal' | 'canary',
  appIcon: string,        // path to app icon
  rendererDir: string,    // path to built renderer HTML
  mainWindowUrl: string,  // URL to load in BrowserWindow
  updaterFeedUrl: string, // URL for auto-update server
}
```
