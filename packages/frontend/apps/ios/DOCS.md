# @affine/ios — Code Documentation

The iOS native app built with Capacitor v7. Wraps the mobile web app with native iOS capabilities: native storage (SQLite via `affine_mobile_native`), haptic feedback, iOS privacy controls, and system browser integration.

---

## How Capacitor works here

```
Web layer (TypeScript)
  @affine/core + @affine/mobile
  ↓ calls via Capacitor plugin bridge
Native layer (Swift)
  Capacitor plugins → affine_mobile_native (UniFFI)
  ↓ returns base64 / token strings
  ↑ decoded by @affine/mobile-shared
Web layer receives clean Uint8Array data
```

---

## App entry (`src/app.tsx`)

Extends `@affine/mobile`'s app with iOS-specific plugin wiring:

```ts
// All mobile modules + mobile router
configureMobileModules(framework);

// Storage: SQLite via affine_mobile_native Capacitor plugin
framework.impl(NbstoreProvider, {
  openStore: async universalId => {
    const dbPath = await getDocStoragePath(universalId);
    await NbstorePlugin.connect({ universalId, path: dbPath });
    return createCapacitorNbstoreAdapter(universalId);
  },
});

// OAuth: opens in Safari (SFSafariViewController)
framework.impl(PopupWindowProvider, {
  open: url => Browser.open({ url }),
});

// Haptics: native iOS haptic engine
framework.impl(HapticProvider, {
  impact: style => Haptics.impact({ style }), // ImpactStyle.Light/Medium/Heavy
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

## iOS-specific plugins

| Plugin                                       | Purpose                                            |
| -------------------------------------------- | -------------------------------------------------- |
| `@capacitor/app`                             | App lifecycle (pause/resume for worker management) |
| `@capacitor/browser`                         | Opens OAuth URLs in SFSafariViewController         |
| `@capacitor/keyboard`                        | Native keyboard show/hide events                   |
| `@capacitor/haptics`                         | Taptic Engine haptic feedback                      |
| `capacitor-plugin-app-tracking-transparency` | iOS ATT privacy dialog                             |

### App Tracking Transparency (ATT)

iOS 14.5+ requires explicit user permission before tracking. The app requests this on first launch:

```ts
import { AppTrackingTransparency } from 'capacitor-plugin-app-tracking-transparency';

const { status } = await AppTrackingTransparency.requestPermission();
// status: 'authorized' | 'denied' | 'notDetermined' | 'restricted'

if (status === 'authorized') {
  track.opt_in_tracking();
} else {
  track.opt_out_tracking();
}
```

---

## Build and sync

```bash
# Sync web build to iOS project
yarn sync

# Sync with dev server (for local development)
CAP_SERVER_URL=http://localhost:8080 yarn sync:dev

# Open Xcode
yarn xcode

# Regenerate TypeScript bindings from Capacitor plugins
yarn codegen
```

The iOS native project lives in `ios/App/`. The Xcode project compiles Swift, links `affine_mobile_native.xcframework`, and embeds the web assets from the `dist/` folder.

---

## Native storage path

On iOS, workspace databases are stored in the app's Documents directory:

```
/var/mobile/Containers/Data/Application/<uuid>/Documents/workspaces/<universalId>.db
```

Files in Documents survive app updates and are backed up to iCloud unless the app opts out.
