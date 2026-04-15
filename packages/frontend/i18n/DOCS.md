# @affine/i18n — Code Documentation

Multi-language support for AFFiNE. Provides 22 locales, a type-safe translation proxy, React hooks, and lazy loading so only the active language is bundled.

---

## Table of Contents

1. [Setup](#1-setup)
2. [Using translations in React](#2-using-translations-in-react)
3. [Using translations outside React](#3-using-translations-outside-react)
4. [The I18n proxy](#4-the-i18n-proxy)
5. [Supported languages](#5-supported-languages)
6. [Adding / updating translations](#6-adding--updating-translations)
7. [Date and time formatting](#7-date-and-time-formatting)

---

## 1. Setup

Initialize once at app startup before rendering:

```ts
import { createI18nWrapper, I18nextProvider } from '@affine/i18n';

// Initialize i18next singleton
const i18n = createI18nWrapper();

// Wrap React tree
<I18nextProvider i18n={i18n}>
  <App />
</I18nextProvider>
```

---

## 2. Using translations in React

```tsx
import { useI18n } from '@affine/i18n';

function MyComponent() {
  const t = useI18n();

  return (
    <div>
      {/* Simple key */}
      <h1>{t['com.affine.workspace.title']()}</h1>

      {/* With interpolation */}
      <p>{t['com.affine.notification.member-count']({ count: 5 })}</p>

      {/* Pluralisation */}
      <span>{t['com.affine.doc.count']({ count: 3 })}</span>
    </div>
  );
}
```

`useI18n()` returns the same `I18n` proxy described below. Re-renders when the active language changes.

---

## 3. Using translations outside React

```ts
import { I18n } from '@affine/i18n';

// Works anywhere — services, utilities, etc.
const message = I18n['com.affine.error.network']();
const title = I18n['com.affine.workspace.title']();
```

`I18n` is a global singleton proxy. It always uses the currently active language.

---

## 4. The I18n proxy

`I18n` is a `Proxy` object. Each property access looks up the key in i18next. Calling the property returns the translated string:

```ts
// Key with no variables
I18n['com.affine.title']()   // → "AFFiNE"

// Key with template variable {{name}}
I18n['com.affine.hello']({ name: 'Alice' })  // → "Hello, Alice"

// In JSX with embedded components (use <Trans> instead)
import { Trans } from '@affine/i18n';
<Trans i18nKey="com.affine.terms.link" components={{ a: <a href="/terms" /> }} />
```

**Type safety:** The generated `i18n.gen.ts` file contains the full union of all valid keys. TypeScript will error if you access a key that doesn't exist in `en.json`.

---

## 5. Supported languages

22 locales. English is bundled; all others are lazy-loaded on first use.

| Code      | Language              | RTL? |
| --------- | --------------------- | ---- |
| `en`      | English               |      |
| `zh-Hans` | Chinese (Simplified)  |      |
| `zh-Hant` | Chinese (Traditional) |      |
| `de`      | German                |      |
| `fr`      | French                |      |
| `es`      | Spanish               |      |
| `ja`      | Japanese              |      |
| `ko`      | Korean                |      |
| `it`      | Italian               |      |
| `pt-BR`   | Portuguese (Brazil)   |      |
| `ru`      | Russian               |      |
| `pl`      | Polish                |      |
| `sv-SE`   | Swedish               |      |
| `uk`      | Ukrainian             |      |
| `ar`      | Arabic                | ✓    |
| `ur`      | Urdu                  | ✓    |
| `fa`      | Persian               | ✓    |
| `nb`      | Norwegian Bokmål      |      |
| `ca`      | Catalan               |      |
| `da`      | Danish                |      |
| `el-GR`   | Greek                 |      |
| `hi`      | Hindi                 |      |

**Fallback chain:** `zh-Hant` → `zh-Hans` → `en`. All others fall back directly to `en`.

---

## 6. Adding / updating translations

### Add a new key

1. Add it to `src/resources/en.json`:
   ```json
   {
     "com.affine.my-feature.title": "My Feature",
     "com.affine.my-feature.count": "{{count}} items"
   }
   ```
2. Run `yarn build` in `packages/frontend/i18n/` — this regenerates `i18n.gen.ts` with the new typed keys.
3. Use the key: `I18n['com.affine.my-feature.title']()`.
4. Submit the English key to the translation platform for other languages.

### Key naming convention

```
com.affine.<feature>.<element>
```

Examples:

- `com.affine.workspaceList.create-workspace` — Create workspace button
- `com.affine.settings.workspace.storage.limitReached` — Storage limit message
- `com.affine.ai.chat.send` — Send button in AI chat

### Update a translation

Edit the value in `en.json` (or the locale-specific JSON). The key stays the same. Run `yarn build` if you changed the key name.

---

## 7. Date and time formatting

```ts
import { i18nTime } from '@affine/i18n';

// Format a date with the current locale
const formatted = i18nTime(new Date(), {
  absolute: { accuracy: 'day' }, // e.g. "April 15, 2026"
});

const relative = i18nTime(new Date(), {
  relative: { accuracy: 'minute' }, // e.g. "2 minutes ago"
});
```

`i18nTime` wraps the `Intl.DateTimeFormat` / `Intl.RelativeTimeFormat` APIs with locale awareness. It automatically uses the currently active i18n language.
