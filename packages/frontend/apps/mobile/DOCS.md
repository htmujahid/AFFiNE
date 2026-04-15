# @affine/mobile — Code Documentation

The mobile Progressive Web App (PWA). Runs in mobile browsers and as a PWA install. Same core as the desktop web app but with touch-optimized layout and mobile-specific implementations.

---

## Entry points

```
src/
  index.html     ← HTML shell with mobile viewport meta
  index.tsx      ← mount <App />
  app.tsx        ← DI setup + React tree
```

---

## Mobile-specific DI wiring

```ts
const framework = new Framework();

configureCommonModules(framework);

// Mobile router (different URL structure, swipe navigation)
configureMobileRouter(framework);

// Mobile workbench (bottom nav, sheet-based layout)
configureMobileWorkbenchModule(framework);

// Haptic feedback via Vibration API
framework.impl(HapticProvider, {
  impact: style => {
    const durations = { light: 10, medium: 20, heavy: 30 };
    navigator.vibrate?.(durations[style] ?? 15);
  },
  notification: () => navigator.vibrate?.([10, 50, 10]),
  selection: () => navigator.vibrate?.(5),
});

// Virtual keyboard height tracking via visualViewport
framework.impl(VirtualKeyboardProvider, {
  onChange: callback => {
    const handler = () => {
      const keyboardHeight = window.innerHeight - (window.visualViewport?.height ?? window.innerHeight);
      callback(Math.max(0, keyboardHeight));
    };
    window.visualViewport?.addEventListener('resize', handler);
    return () => window.visualViewport?.removeEventListener('resize', handler);
  },
});
```

---

## Mobile router

`@affine/core/mobile/router` uses a different URL structure optimized for mobile navigation:

- Back/forward gestures map to browser history
- Sheet-based navigation (bottom sheets slide up instead of full-page transitions)
- No multi-tab interface (one document at a time)

---

## Worker setup

Same nbstore worker strategy as the web app. SharedWorker is used if the browser supports it (Safari on iOS ≥ 16.4+). Falls back to a dedicated Worker.

---

## PWA configuration

The `index.html` includes mobile-specific meta tags:

- `viewport` with `viewport-fit=cover` for notch/safe-area support
- `theme-color` for status bar styling
- `apple-mobile-web-app-capable` for iOS standalone mode
- `manifest.json` for PWA install prompt
