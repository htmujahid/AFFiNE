# @affine/electron-renderer — Code Documentation

The Electron renderer process — the UI layer of the desktop app. Nearly identical to `@affine/web` but wired with desktop-specific implementations.

---

## Entry points

```
src/
  index.tsx           ← mount <App /> in #app
  popup/index.tsx     ← entry for popup windows (OAuth, modals)
  app/
    app.tsx           ← DI setup + React tree
    effects.ts        ← configures the DI framework
    theme-sync.ts     ← OS theme → AFFiNE theme
    language-sync.ts  ← OS language → i18n locale
  background-worker/  ← Web Worker for background tasks
  shell/              ← Shell chrome (window controls)
```

---

## DI differences from the web app

The desktop app adds or replaces several implementations:

```ts
// Desktop workbench (saves/restores view state to disk)
configureDesktopWorkbenchModule(framework);

// SQLite via helper process (not IndexedDB)
framework.impl(NbstoreProvider, DesktopNbstoreProvider);

// Electron IPC bridge (not window.open)
framework.impl(PopupWindowProvider, ElectronPopupWindowProvider);

// Desktop API for Electron-specific operations
framework.impl(DesktopApiService, ElectronDesktopApiService);

// Persistent state via electron-store (not localStorage)
configureElectronStateStorageImpls(framework);
```

---

## OS synchronization

### Theme sync (`app/theme-sync.ts`)

Listens to Electron's `nativeTheme.themeSource` and keeps AFFiNE's theme in sync:

```ts
// When OS switches dark/light → update ThemeService
nativeTheme.on('updated', () => {
  themeService.theme$.next(nativeTheme.shouldUseDarkColors ? 'dark' : 'light');
});
```

### Language sync (`app/language-sync.ts`)

On startup, reads the OS locale and sets i18n if the user hasn't overridden it:

```ts
const osLocale = app.getLocale(); // e.g. 'zh-CN'
if (!userHasExplicitLanguage) {
  i18n.changeLanguage(mapOsLocaleToAffine(osLocale));
}
```

---

## Windows-specific shell (`shell/`)

On Windows, AFFiNE renders its own window title bar (no native chrome):

- Custom minimize, maximize, close buttons
- Drag region for moving the window
- Menu integration

These are enabled/disabled based on `BUILD_CONFIG.isWindows`.

---

## Popup windows

The `popup/index.tsx` entry is a separate HTML page loaded in Electron's popup `BrowserWindow`. Used for:

- OAuth callbacks (Google, GitHub sign-in)
- Large modals that need their own window context

The popup communicates back to the main window via `window.opener.postMessage()`.
