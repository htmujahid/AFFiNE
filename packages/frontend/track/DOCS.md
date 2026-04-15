# @affine/track — Code Documentation

Type-safe analytics and telemetry system. Provides a 200+ event schema, session management, a middleware pipeline, auto-tracking via HTML data attributes, and Sentry error integration.

---

## Table of Contents

1. [Tracking an event](#1-tracking-an-event)
2. [Auto-tracking with data attributes](#2-auto-tracking-with-data-attributes)
3. [Session and user identity](#3-session-and-user-identity)
4. [Middleware pipeline](#4-middleware-pipeline)
5. [Telemetry transport](#5-telemetry-transport)
6. [Event schema reference](#6-event-schema-reference)

---

## 1. Tracking an event

```ts
import { track } from '@affine/track';

// Simple event with no arguments
track.$.header.actions.createDoc();

// Event with typed arguments
track.$.workspace.header.createWorkspace({ flavour: 'affine-cloud' });

// From a specific page context
track.allDocsPage.docList.$.openDoc({ mode: 'page' });
```

**The four-level hierarchy:** `[page].[segment].[module].[event]`

- `$` means "any context" (the page/segment/module is unknown or generic).
- All four levels are required but `$` can fill any position.
- TypeScript enforces that arguments match the event's declared schema.

---

## 2. Auto-tracking with data attributes

For components where you can't easily call `track.*()` in JavaScript, add HTML data attributes:

```tsx
// Button that tracks a click event
<button
  data-event-props="$.navigation.quickSearch.open"
>
  Open
</button>

// With arguments
<button
  data-event-props="$.editor.toolbar.bold"
  data-event-args-type="heading"
>
  Bold
</button>

// Or with a single arg key
<div
  data-event-props="$.workspace.list.$.createWorkspace"
  data-event-arg="affine-cloud"
>
```

`enableAutoTrack(rootElement, trackFn)` sets up a `click` listener on the element and reads these attributes automatically.

---

## 3. Session and user identity

```ts
import { track } from '@affine/track';

// Identify the current user after sign-in
track.identify('user-id-123');

// Clear identity on sign-out
track.reset();

// Opt in/out
track.opt_in_tracking();
track.opt_out_tracking();
console.log(track.has_opted_in_tracking());
```

**Session tracking:**

- Sessions are auto-detected from localStorage/sessionStorage.
- A new session starts after 30 minutes of inactivity.
- `sessionNumber` increments each session (used for funnel analysis).
- Page visibility changes are tracked for engagement time.

---

## 4. Middleware pipeline

Enrich every event with additional properties before it's sent:

```ts
import { setTelemetryTransport } from '@affine/track';

// Add a middleware that appends workspace context to all events
track.middleware((eventName, properties) => ({
  workspaceId: currentWorkspaceId,
  workspaceFlavour: currentFlavour,
}));
```

Multiple middlewares compose — each return value is merged into the event properties.

---

## 5. Telemetry transport

By default, events are queued but not sent anywhere. Inject a transport to actually send them:

```ts
import { setTelemetryTransport } from '@affine/track';

setTelemetryTransport({
  setContext(context) {
    // Called once at startup with app/user context
    posthog.register({
      appVersion: context.appVersion,
      distribution: context.distribution,
    });
  },

  track(event) {
    posthog.capture(event.eventName, event.params);
  },

  pageview(event) {
    posthog.capture('$pageview', { url: event.params.url });
  },

  async flush() {
    await posthog.flush();
    return { count: 0 }; // TelemetryAck
  },
});
```

**`TelemetryEvent` shape:**

```ts
interface TelemetryEvent {
  eventName: string;
  params: Record<string, unknown>;
  userId: string | null;
  userProperties: Record<string, unknown>;
  clientId: string;
  sessionId: string;
  eventId: string;
  timestampMicros: number;
  context: {
    appVersion: string;
    environment: 'production' | 'development' | 'test';
    distribution: 'browser' | 'desktop' | 'mobile';
    channel: 'canary' | 'beta' | 'stable' | 'internal';
    locale: string;
    timezone: string;
    url: string;
    referrer: string;
  };
}
```

---

## 6. Event schema reference

Events are grouped by page and segment. A selection of the most important:

### App & Navigation

| Event path                    | Arguments  | When                      |
| ----------------------------- | ---------- | ------------------------- |
| `$.app.$.checkUpdates`        | —          | Check for updates clicked |
| `$.app.$.downloadUpdate`      | —          | Update download started   |
| `$.navigation.$.openInNewTab` | `{ type }` | Doc opened in new tab     |
| `$.navigation.$.navigate`     | `{ to }`   | Route changed             |

### Documents

| Event path                   | Arguments  | When                   |
| ---------------------------- | ---------- | ---------------------- |
| `$.allDocs.$.createDoc`      | —          | New doc created        |
| `$.allDocs.$.openDoc`        | `{ mode }` | Doc opened             |
| `$.allDocs.$.deleteDoc`      | `{ type }` | Doc deleted            |
| `$.allDocs.$.renameDoc`      | —          | Doc renamed            |
| `$.allDocs.$.switchPageMode` | `{ mode }` | Switched page/edgeless |

### Workspace

| Event path                           | Arguments     | When                       |
| ------------------------------------ | ------------- | -------------------------- |
| `$.workspace.$.createWorkspace`      | `{ flavour }` | Workspace created          |
| `$.workspace.$.upgradeWorkspace`     | —             | Workspace upgraded to team |
| `$.workspace.$.enableCloudWorkspace` | —             | Local → cloud              |

### Editor

| Event path                       | Arguments | When                  |
| -------------------------------- | --------- | --------------------- |
| `$.editor.toolbar.bold`          | —         | Bold applied          |
| `$.editor.toolbar.italic`        | —         | Italic applied        |
| `$.editor.toolbar.strikeThrough` | —         | Strikethrough applied |
| `$.editor.edgeless.foldNote`     | —         | Edgeless note folded  |

### Cloud / Sharing

| Event path                       | Arguments             | When                 |
| -------------------------------- | --------------------- | -------------------- |
| `$.sharePanel.$.createShareLink` | `{ type }`            | Share link created   |
| `$.sharePanel.$.copyShareLink`   | —                     | Share link copied    |
| `$.paywall.$.viewPlans`          | `{ plan, recurring }` | Plans page viewed    |
| `$.paywall.$.subscribe`          | `{ plan, recurring }` | Subscription started |

### AI

| Event path                   | Arguments  | When                    |
| ---------------------------- | ---------- | ----------------------- |
| `$.aiChat.$.sendMessage`     | `{ type }` | Message sent to AI      |
| `$.aiChat.$.addEmbeddingDoc` | —          | Doc added to AI context |

### Comments

| Event path                   | Arguments | When             |
| ---------------------------- | --------- | ---------------- |
| `$.docPage.$.createComment`  | —         | Comment created  |
| `$.docPage.$.resolveComment` | —         | Comment resolved |
| `$.docPage.$.deleteComment`  | —         | Comment deleted  |
