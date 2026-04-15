# @affine/admin — Code Documentation

Self-hosted admin panel for managing AFFiNE server instances. A standalone React app (separate from the main workspace app) with its own routing, data layer, and UI framework.

---

## Table of Contents

1. [What it is and isn't](#1-what-it-is-and-isnt)
2. [Authentication and access control](#2-authentication-and-access-control)
3. [Module reference](#3-module-reference)
4. [Data fetching pattern](#4-data-fetching-pattern)
5. [UI components](#5-ui-components)
6. [Key hooks](#6-key-hooks)

---

## 1. What it is and isn't

**Is:**

- A React Router v7 SPA serving at `/admin/*`
- Direct GraphQL consumer (no `@affine/core` DI system)
- Uses `shadcn/ui` + Tailwind CSS (completely isolated from the main app's Vanilla Extract styles)
- Manages server config, accounts, workspaces, job queues, AI settings

**Is not:**

- A document editor
- A workspace collaboration tool
- Part of the `@affine/core` module system

---

## 2. Authentication and access control

The admin panel checks for an admin role before rendering any content:

```
/ (root)
  Check: is the server initialized?
  No  → /admin/setup   (first-time wizard)
  Yes ↓

  Check: is user logged in and has admin role?
  No  → /admin/auth   (login page)
  Yes ↓

  <Layout> with breadcrumb navigation
  → nested route content
```

The `useCurrentUser()` hook drives this check. Admin role comes from the `UserFeature` system on the server.

---

## 3. Module reference

```
src/modules/
  auth/          Login page, session management
  setup/         First-run server setup wizard
  dashboard/     Overview stats (users, workspaces, storage)
  accounts/      User account list, search, disable/enable
  workspaces/    Workspace list, quota management
  queue/         BullMQ job queue monitor (@queuedash/ui)
  ai/            AI provider config, quotas, model settings
  settings/      Server-wide settings (SMTP, OAuth, storage)
  about/         Version info, license
```

### `auth/`

Sign-in form. Redirects to `/admin/dashboard` after successful login. Supports email/password and OAuth (if configured).

### `setup/`

Multi-step first-run wizard:

1. Check if server already initialized → redirect if so
2. Admin account creation
3. Server URL configuration
4. Optional: SMTP / OAuth setup

### `dashboard/`

Aggregate stats pulled from GraphQL:

- Total users, active users in last 30 days
- Total workspaces, total storage used
- Recent sign-ups chart

### `accounts/`

Full user account management:

- List with pagination, search by email/name
- View user details (quota, features, sessions)
- Enable/disable account
- Assign/revoke admin role
- Reset password (triggers email)

### `workspaces/`

Workspace management:

- List with search by name/ID
- View workspace members and their roles
- Adjust per-workspace quotas
- Delete workspace

### `queue/`

Embeds `@queuedash/ui` pointed at the BullMQ dashboard API. Shows:

- Queue names and job counts (active, waiting, completed, failed)
- Job details and retry
- Pause/resume queues

### `ai/`

AI provider configuration:

- Enable/disable AI globally
- Configure model providers (API keys, endpoints)
- Set per-workspace AI quotas
- View token usage stats

### `settings/`

Server-wide configuration, organized into tabs:

- **General**: server name, external URL
- **Account**: sign-up mode (open/invite-only), OAuth providers
- **Storage**: local vs S3 config, max blob size
- **SMTP**: email server config for transactional email

---

## 4. Data fetching pattern

The admin panel uses **SWR** with a custom GraphQL integration (not the `@affine/graphql` package).

### `useQuery` — fetch with SWR

```ts
import { useQuery } from './use-graphql';

function AccountsPage() {
  const { data, isLoading, error } = useQuery(GetUsersDocument, {
    take: 20,
    skip: page * 20,
  });

  if (isLoading) return <Skeleton />;
  return <UserTable users={data.users} />;
}
```

### `useMutation` — mutate with optimistic UI

```ts
const [updateQuota, { loading }] = useMutation(UpdateWorkspaceQuotaMutation);

await updateQuota({ workspaceId, storageQuota: newQuota });
```

### `useQueryImmutable` — cache-first reads

For data that rarely changes (server config, current user):

```ts
const { data: serverConfig } = useQueryImmutable(GetServerConfigDocument);
```

### `useQueryInfinite` — paginated lists

```ts
const { data, fetchMore } = useQueryInfinite(
  GetUsersDocument,
  { take: 20 },
  {
    getNextPageParam: lastPage => lastPage.users.nextCursor,
  }
);
```

---

## 5. UI components

Uses `shadcn/ui` components (Radix UI + Tailwind). Located in `src/components/ui/`:

All standard shadcn components: `Button`, `Card`, `Dialog`, `Input`, `Select`, `Table`, `Badge`, `Tabs`, `Toast`, `Tooltip`, `Form`, `Separator`, `ScrollArea`, `Sheet`, `Command`, `Popover`, `Avatar`, `Progress`, `Switch`, `Checkbox`, `RadioGroup`, `Textarea`, `Label`, `Skeleton`, `Collapsible`, `Alert`, `AlertDialog`, `DropdownMenu`, `HoverCard`, `NavigationMenu`, `Pagination`, `Resizable`, `Breadcrumb`, `Carousel`, `Drawer`.

---

## 6. Key hooks

```ts
// Current logged-in user + admin check
const { user, isAdmin } = useCurrentUser();

// Server-wide config (URL, features, auth methods)
const serverConfig = useServerConfig();

// Responsive layout
const isMobile = useMediaQuery('(max-width: 768px)');
```

**`useServerConfig()`** returns the `ServerRuntimeConfig` from GraphQL:

- `name` — server display name
- `externalUrl` — public URL
- `enableTelemetry`
- `credentialsRequirement` — password policy
- `oauthProviders` — enabled OAuth providers
- `features` — enabled server features
