# @affine/android — Code Documentation

The Android native app built with Capacitor v7. Mirrors the iOS app but with Android-specific plugins: in-app browser for OAuth, status bar control, and different dev server addressing.

---

## App entry (`src/app.tsx`)

Extends `@affine/mobile`'s app with Android-specific plugin wiring:

```ts
// Storage: SQLite via affine_mobile_native Capacitor plugin
framework.impl(NbstoreProvider, {
  openStore: async universalId => {
    const dbPath = await getDocStoragePath(universalId);
    await NbstorePlugin.connect({ universalId, path: dbPath });
    return createCapacitorNbstoreAdapter(universalId);
  },
});

// OAuth: opens in WebView (InAppBrowser, not system browser)
framework.impl(PopupWindowProvider, {
  open: (url, target) => InAppBrowser.openWebView({ url }),
});

// Haptics: Android vibration
framework.impl(HapticProvider, {
  impact: style => Haptics.impact({ style }),
  notification: type => Haptics.notification({ type }),
  selection: () => Haptics.selectionChanged(),
});

// Virtual keyboard: Capacitor Keyboard plugin
framework.impl(VirtualKeyboardProvider, {
  onChange: callback => {
    Keyboard.addListener('keyboardWillShow', ({ keyboardHeight }) => callback(keyboardHeight));
    Keyboard.addListener('keyboardWillHide', () => callback(0));
  },
});
```

---

## Android-specific plugins

| Plugin                  | Purpose                      | iOS equivalent                |
| ----------------------- | ---------------------------- | ----------------------------- |
| `@capacitor/app`        | App lifecycle (pause/resume) | Same                          |
| `@capacitor/keyboard`   | Keyboard events              | Same                          |
| `@capacitor/haptics`    | Vibration feedback           | Same                          |
| `@capacitor/status-bar` | Status bar color/style       | —                             |
| `@capgo/inappbrowser`   | OAuth in WebView             | `@capacitor/browser` (Safari) |

### Status bar

Android allows controlling the status bar color and text:

```ts
import { StatusBar, Style } from '@capacitor/status-bar';

// Match the app theme
StatusBar.setStyle({ style: isDark ? Style.Dark : Style.Light });
StatusBar.setBackgroundColor({ color: '#000000' });
```

### OAuth browser difference

iOS uses Safari (`SFSafariViewController`) for OAuth — this is preferred because it shares the Safari cookie jar. Android uses `@capgo/inappbrowser` (a WebView) because Capacitor's `Browser` plugin on Android doesn't reliably handle deep-link callbacks.

---

## Build and sync

```bash
# Sync web build to Android project
yarn sync

# Sync with dev server
# Note: 10.0.2.2 is the Android emulator's alias for localhost
CAP_SERVER_URL=http://10.0.2.2:8080 yarn sync:dev

# Open Android Studio
yarn studio
```

The Android project lives in `android/`. Gradle compiles Kotlin, links `affine_mobile_native.aar`, and packages the web assets from `dist/`.

---

## Key differences from iOS

|                  | iOS                                     | Android               |
| ---------------- | --------------------------------------- | --------------------- |
| OAuth browser    | SFSafariViewController (shares cookies) | InAppBrowser WebView  |
| Privacy consent  | ATT dialog required                     | Not required          |
| Dev server       | `localhost` / local IP                  | `10.0.2.2` (emulator) |
| App store        | Apple App Store                         | Google Play Store     |
| Native extension | `.xcframework`                          | `.aar`                |

---

## Native storage path

On Android, workspace databases are stored in the app's internal files directory:

```
/data/data/<package>/files/workspaces/<universalId>.db
```

This directory is private to the app and not accessible to other apps without root.
