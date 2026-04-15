# @affine/web — Code Documentation

The desktop browser SPA served at `app.affine.pro`. Bundles `@affine/core` with web-specific storage, routing, and worker configuration.

---

## Entry points

```
src/
  index.html      ← HTML shell
  index.tsx       ← mount <App /> in #app
  app.tsx         ← DI setup + React tree
```

---

## DI framework setup (`app.tsx`)

The app wires `@affine/core` modules with browser-specific implementations:

```ts
const framework = new Framework();

// 1. All shared core modules
configureCommonModules(framework);

// 2. Desktop router and workbench
configureBrowserWorkbenchModule(framework);
configureDesktopRouter(framework);

// 3. Storage implementations
configureLocalStorageStateStorageImpls(framework);

// 4. Browser-specific provider implementations
framework.impl(NbstoreProvider, BrowserNbstoreProvider);
framework.impl(PopupWindowProvider, BrowserPopupWindowProvider);
framework.impl(GlobalDialogService, BrowserGlobalDialogService);
```

---

## Worker strategy

`NbstoreProvider` coordinates nbstore across browser tabs:

```
Multiple browser tabs open
  ↓
Try SharedWorker (one shared connection across all tabs)
  → If unavailable (Safari, disabled) → dedicated Worker per tab

Worker resumes on: window focus, click
Worker pauses on:  window blur (tab switch)
Worker disposes on: beforeunload
```

Why pause on blur? The nbstore worker does background sync. Pausing it when the tab is hidden reduces battery and CPU usage on mobile.

---

## OAuth redirect proxy

OAuth callbacks go through `/redirect-proxy` — a minimal page that passes the OAuth code back to the opener window:

```ts
framework.impl(PopupWindowProvider, {
  open: (url, target) => window.open(url, target, 'width=600,height=700'),
});
```

The popup URL is `/redirect-proxy?redirect_uri=...&code=...`. The proxy reads the code and posts a message to `window.opener`.

---

## Router

Uses React Router v6 with the desktop route config from `@affine/core/desktop/router`. This router includes:

- `/` → workspace selector / redirect to last workspace
- `/workspace/:workspaceId/*` → workspace pages
- `/sign-in`, `/sign-up` → auth pages
- `/invite/:inviteId` → workspace invite
- `/redirect-proxy` → OAuth callback handler

---

## Key differences from other app packages

| Feature     | Web                           | Electron               | Mobile                   |
| ----------- | ----------------------------- | ---------------------- | ------------------------ |
| Storage     | IndexedDB via Worker          | SQLite via native      | IndexedDB via Worker     |
| Worker type | SharedWorker (when available) | Helper process (IPC)   | Dedicated Worker         |
| Window APIs | `window.open()` for OAuth     | `ipcRenderer.invoke()` | Capacitor Browser plugin |
| Router      | Desktop router                | Desktop router         | Mobile router            |
| Updates     | Manual page refresh           | electron-updater       | PWA service worker       |
