# @affine/component — Code Documentation

AFFiNE's design system. All shared UI components, hooks, and the Lit-to-React bridge live here. Every frontend app and `@affine/core` imports from this package.

---

## Table of Contents

1. [Package structure](#1-package-structure)
2. [UI components](#2-ui-components)
3. [Composite components](#3-composite-components)
4. [Hooks](#4-hooks)
5. [Utilities](#5-utilities)
6. [Lit-to-React bridge](#6-lit-to-react-bridge)
7. [Theming](#7-theming)
8. [Styling approach](#8-styling-approach)

---

## 1. Package structure

```
src/
  ui/                  ← 36 primitive UI components
  components/          ← 12+ composite/domain components
  hooks/               ← Custom React hooks
  utils/               ← DOM utilities
  lit-react/           ← Lit Web Component interop
  theme/               ← Theme config
  styles/              ← Global CSS reset / base styles
```

---

## 2. UI components

All live under `src/ui/`. Import path: `@affine/component/ui/<name>`.

### Button

```tsx
import { Button, IconButton } from '@affine/component';

<Button variant="primary" size="default" loading={false} disabled={false}>
  Click me
</Button>

<IconButton icon={<PlusIcon />} tooltip="Add item" />
```

**`variant`:** `'primary' | 'secondary' | 'plain' | 'error' | 'success' | 'custom'`  
**`size`:** `'default' | 'large' | 'extraLarge' | 'custom'`

---

### Input

```tsx
import { Input, RowInput } from '@affine/component';

// Standard input
<Input
  value={value}
  onChange={setValue}
  placeholder="Type here..."
  endFix={<ClearButton />}
/>

// Inline editable row (for doc titles, tag names)
<RowInput defaultValue="Title" onConfirm={save} />
```

---

### Modal family

```tsx
import { Modal, ConfirmModal, PromptModal } from '@affine/component';

// Base modal — full control
<Modal open={open} onOpenChange={setOpen} title="Settings">
  {children}
</Modal>;

// Confirm dialog with Yes/No
const confirmed = await openConfirmModal({
  title: 'Delete workspace?',
  description: 'This cannot be undone.',
  confirmText: 'Delete',
  confirmButtonOptions: { variant: 'error' },
});

// Prompt for text input
const value = await openPromptModal({
  title: 'Rename',
  label: 'New name',
  initialValue: current,
});
```

---

### Menu

```tsx
import { Menu, MenuItem, MenuSeparator, MenuTrigger } from '@affine/component';

<Menu
  items={
    <>
      <MenuItem onClick={rename} prefixIcon={<EditIcon />}>
        Rename
      </MenuItem>
      <MenuSeparator />
      <MenuItem onClick={del} type="danger">
        Delete
      </MenuItem>
    </>
  }
>
  <MenuTrigger>Options</MenuTrigger>
</Menu>;
```

Backed by Radix UI `DropdownMenu`. Supports sub-menus and keyboard navigation.

---

### Tooltip

```tsx
import { Tooltip } from '@affine/component';

<Tooltip content="This is a tooltip" side="top">
  <IconButton icon={<InfoIcon />} />
</Tooltip>;
```

---

### Notification (Toast)

```tsx
import { notify } from '@affine/component';

notify.success({ title: 'Saved', message: 'Changes were saved.' });
notify.error({ title: 'Error', message: 'Failed to save.' });
notify.warning({ title: 'Warning', message: 'Disk space low.' });
notify({
  title: 'Custom',
  action: { label: 'Undo', onClick: undo },
  duration: 5000,
});
```

Backed by `sonner`. Use the `notify` function directly — don't render `<Sonner />` yourself (it's mounted at the app level via `NotificationCenter`).

---

### Other primitives

| Component        | Import                     | Notes                                    |
| ---------------- | -------------------------- | ---------------------------------------- |
| `Avatar`         | `@affine/component`        | Renders user avatar or initials fallback |
| `Tabs`, `Tab`    | `@affine/component`        | Tab navigation bar                       |
| `Checkbox`       | `@affine/component`        | Accessible checkbox                      |
| `Slider`         | `@affine/component`        | Range input                              |
| `DatePicker`     | `@affine/component`        | Calendar date picker                     |
| `Table`          | `@affine/component`        | Data table with sorting                  |
| `Masonry`        | `@affine/component`        | Masonry grid layout                      |
| `Scrollable`     | `@affine/component`        | Styled scrollable container              |
| `Popover`        | `@affine/component`        | Floating content popover                 |
| `DraggablePanel` | `@affine/component/ui/dnd` | Resizable draggable panel                |
| `AudioPlayer`    | `@affine/component`        | Audio file playback                      |

---

## 3. Composite components

Larger, domain-aware components that combine primitives:

| Component            | Path                             | Purpose                                  |
| -------------------- | -------------------------------- | ---------------------------------------- |
| `AvatarStack`        | `components/avatar-stack`        | Show multiple user avatars               |
| `AuthComponents`     | `components/auth-components`     | Login / sign-up form fields              |
| `NotificationCenter` | `components/notification-center` | Toast container (mount once at root)     |
| `ProviderComposer`   | `components/provider-composer`   | Compose multiple React context providers |
| `AffineBanner`       | `components/affine-banner`       | Top-of-page announcement banner          |
| `Card`               | `components/card`                | Standard card container                  |
| `DocCard`            | `components/doc-card`            | Doc list card with preview               |
| `WorkspaceAvatar`    | `components/workspace-avatar`    | Workspace icon with fallback             |
| `PageListSkeleton`   | `components/page-list`           | Loading skeleton for doc lists           |
| `PropertyTable`      | `components/property-table`      | Key-value property rows (doc info)       |
| `TagItem`            | `components/tag`                 | Colored tag chip                         |
| `MemberListItem`     | `components/member-list`         | Workspace member row                     |

---

## 4. Hooks

**File:** `src/hooks/`

```ts
import { useAutoFocus, useAutoSelect, useDisposable, useRefEffect, useThemeColorMeta, useThemeValue } from '@affine/component';
```

| Hook                | Signature              | Purpose                                                 |
| ------------------- | ---------------------- | ------------------------------------------------------- |
| `useAutoFocus`      | `(ref, condition?)`    | Focus element on mount (or when condition becomes true) |
| `useAutoSelect`     | `(ref)`                | Select all text in input on focus                       |
| `useDisposable`     | `() => Disposables`    | Returns a `Disposables` object for managing cleanups    |
| `useRefEffect`      | `(effect, deps)`       | Like `useEffect` but fires when ref value changes       |
| `useThemeColorMeta` | `() => ThemeColorMeta` | Returns current theme color palette metadata            |
| `useThemeValue`     | `(token) => string`    | Read a single design token value                        |

---

## 5. Utilities

**File:** `src/utils/`

```ts
import { observeIntersection, observeResize, startScopedViewTransition, withUnit } from '@affine/component/utils';
```

| Utility                     | Signature                            | Purpose                                                |
| --------------------------- | ------------------------------------ | ------------------------------------------------------ |
| `observeIntersection`       | `(el, callback, options?)` → cleanup | Wraps IntersectionObserver, returns unobserve function |
| `observeResize`             | `(el, callback)` → cleanup           | Wraps ResizeObserver                                   |
| `startScopedViewTransition` | `(callback, classNames?)`            | View Transition API with cleanup                       |
| `withUnit`                  | `(value, unit)` → string             | Formats CSS values: `withUnit(12, 'px')` → `'12px'`    |

---

## 6. Lit-to-React bridge

**File:** `src/lit-react/`

BlockSuite uses Lit Web Components. This bridge lets you embed them inside React without manual DOM manipulation.

```ts
import { createReactComponentFromLit } from '@affine/component/lit-react';
import { MyLitElement } from '@blocksuite/affine';

// Creates a React component that renders the Lit element
const MyReactComponent = createReactComponentFromLit({
  react: React,
  elementClass: MyLitElement,
});

// Use like any React component
<MyReactComponent
  prop1="value"
  onMyEvent={handler}
/>
```

**How it works internally:**

1. Creates a real DOM element via `customElements.define` if not yet registered.
2. Mounts the Lit element into a `div` container.
3. Forwards React props as Lit element properties.
4. Converts Lit `@event` → React `onEvent` conventions.
5. Cleans up on React unmount.

This is the **only** supported way to use BlockSuite components inside React code. Never instantiate Lit elements directly in React render functions.

---

## 7. Theming

AFFiNE supports light, dark, and system themes. The theme system works through CSS variables.

**CSS variables** are provided by `@toeverything/theme` and applied by `ThemeProvider`. Components use these tokens:

```css
/* Example usage in a component's .css.ts file */
color: var(--affine-text-primary-color);
background: var(--affine-background-primary-color);
border-color: var(--affine-border-color);
```

**Reading tokens in JS:**

```ts
import { useThemeValue } from '@affine/component';

// Get any CSS variable value
const primaryColor = useThemeValue('--affine-primary-color');
```

**Switching themes:**

```ts
import { useService } from '@toeverything/infra';
import { ThemeService } from '@affine/core';

const themeService = useService(ThemeService);
themeService.theme$.next('dark'); // 'light' | 'dark' | 'system'
```

---

## 8. Styling approach

Three styling methods are used — each for different scenarios:

### Vanilla Extract (`.css.ts`) — preferred for component styles

Zero-runtime CSS. Styles are extracted at build time:

```ts
// button.css.ts
import { style } from '@vanilla-extract/css';

export const buttonRoot = style({
  padding: '8px 16px',
  borderRadius: 8,
  ':hover': { opacity: 0.9 },
});
```

```tsx
// button.tsx
import { buttonRoot } from './button.css';
<button className={buttonRoot} />;
```

### Emotion — for dynamic styles that depend on runtime values

```tsx
import { css } from '@emotion/react';

<div
  css={css`
    background: ${themeColor};
  `}
/>;
```

### Inline styles — only for truly dynamic single values

```tsx
<div style={{ '--custom-height': `${height}px` } as CSSProperties} />
```
