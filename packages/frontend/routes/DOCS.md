# @affine/routes — Code Documentation

Centralized route definitions and a `lazy()` loader for code splitting. Used by the admin panel and referenced by core for typed link generation.

---

## Route constants

```ts
import { ROUTES, RELATIVE_ROUTES, FACTORIES } from '@affine/routes';

// Absolute paths
ROUTES.index; // '/'
ROUTES.admin.index; // '/admin'
ROUTES.admin.auth; // '/admin/auth'
ROUTES.admin.setup; // '/admin/setup'
ROUTES.admin.dashboard; // '/admin/dashboard'
ROUTES.admin.accounts; // '/admin/accounts'
ROUTES.admin.workspaces; // '/admin/workspaces'
ROUTES.admin.queue; // '/admin/queue'
ROUTES.admin.ai; // '/admin/ai'
ROUTES.admin.settings.index; // '/admin/settings'
ROUTES.admin.settings.module; // '/admin/settings/:module'
ROUTES.admin.about; // '/admin/about'
ROUTES.admin.notFound; // '/admin/404'

// Relative paths (for nested routes)
RELATIVE_ROUTES.admin.index; // 'admin'
RELATIVE_ROUTES.admin.settings.module; // 'settings/:module'
```

## Factory functions (generate concrete paths)

```ts
// Typed link generators
FACTORIES.home(); // → '/'
FACTORIES.admin(); // → '/admin'
FACTORIES.admin.auth(); // → '/admin/auth'
FACTORIES.admin.dashboard(); // → '/admin/dashboard'
FACTORIES.admin.settings.module({ module: 'general' }); // → '/admin/settings/general'
```

Use factories in `<Link>` components to avoid hardcoding strings:

```tsx
import { FACTORIES } from '@affine/routes';
import { Link } from 'react-router-dom';

<Link to={FACTORIES.admin.settings.module({ module: 'ai' })}>AI Settings</Link>;
```

---

## `lazy()` — code-split component loader

```ts
import { lazy } from '@affine/routes';

// Lazy-load a default export
const DashboardPage = lazy(() => import('./pages/dashboard'));

// Lazy-load a named export
const AccountsTable = lazy(() => import('./components/accounts-table'), 'AccountsTable');

// With a loading fallback
const WorkspacesPage = lazy(
  () => import('./pages/workspaces'),
  undefined,
  <PageSkeleton />
);
```

The returned component wraps itself in `<React.Suspense>` automatically. No need to add `<Suspense>` at the route level.

### Type signature

```ts
function lazy<T extends keyof typeof module = 'default'>(factory: () => Promise<Record<T, ComponentType<any>>>, exportName?: T, fallback?: ReactNode): ComponentType<ComponentProps<(typeof module)[T]>>;
```
