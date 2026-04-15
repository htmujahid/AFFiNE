# @affine/server — Code Documentation

NestJS backend that powers AFFiNE Cloud and self-hosted instances. Handles authentication, real-time document sync, blob storage, AI copilot, payments, and serves the frontend.

---

## Table of Contents

1. [How the server starts](#1-how-the-server-starts)
2. [Deployment flavors](#2-deployment-flavors)
3. [Module anatomy](#3-module-anatomy)
4. [Base layer — infrastructure primitives](#4-base-layer--infrastructure-primitives)
5. [Models layer — database access](#5-models-layer--database-access)
6. [Core modules — business logic](#6-core-modules--business-logic)
7. [Plugin modules — optional features](#7-plugin-modules--optional-features)
8. [Document sync deep-dive](#8-document-sync-deep-dive)
9. [AI Copilot deep-dive](#9-ai-copilot-deep-dive)
10. [Authentication flow](#10-authentication-flow)
11. [Database schema overview](#11-database-schema-overview)
12. [Config system](#12-config-system)
13. [Background jobs](#13-background-jobs)
14. [Key patterns](#14-key-patterns)

---

## 1. How the server starts

```
src/index.ts
  ├── env.flavors.script  → src/cli.ts       (data-migration scripts)
  └── (else)              → src/server.ts    (HTTP/WS server)
```

`src/server.ts` calls `NestFactory.create(AppModule)` and wires up:

- Cookie parser
- CORS (per-request origin resolution, credentials allowed)
- GraphQL file upload middleware (100 MB limit)
- Global `AuthGuard` and `CloudThrottlerGuard`
- Global `CacheInterceptor` and `GlobalExceptionFilter`
- `SocketIoAdapter` (Redis-backed, multi-instance safe)
- Swagger UI at `/api/docs` in dev mode

The `AppModule` itself is **dynamically built** by `buildAppModule(env)` in `src/app.module.ts` — which modules are imported depends on the `SERVER_FLAVOR` environment variable.

---

## 2. Deployment flavors

Controlled by `SERVER_FLAVOR` env var. Defined in `src/env.ts`.

| Flavor               | What it runs                                    | Use case               |
| -------------------- | ----------------------------------------------- | ---------------------- |
| `allinone` (default) | Everything                                      | Local dev, self-hosted |
| `graphql`            | GraphQL API, auth, workspaces, copilot, payment | Cloud API nodes        |
| `sync`               | WebSocket doc sync gateway only                 | Cloud sync nodes       |
| `renderer`           | Doc-to-HTML renderer only                       | Cloud rendering nodes  |
| `doc`                | Doc storage read/write service                  | Cloud doc nodes        |
| `front`              | Static frontend files server                    | Cloud CDN-bypass nodes |
| `script`             | CLI scripts (data migration)                    | DevOps tasks           |

The `AppModuleBuilder` accumulates modules conditionally:

```ts
// src/app.module.ts
factor
  .use(...FunctionalityModules)          // always loaded
  .useIf(() => env.flavors.graphql, GqlModule, WorkspaceModule, CopilotModule, ...)
  .useIf(() => env.flavors.sync || env.flavors.front, SyncModule, TelemetryModule)
  .useIf(() => env.dev || env.selfhosted, WorkerModule, SelfhostModule)
```

---

## 3. Module anatomy

The source tree has four distinct layers:

```
src/
  base/          # Infrastructure — no business logic
  models/        # Typed DB access wrappers (one file per Prisma model)
  core/          # Business modules always loaded when server runs
  plugins/       # Optional features gated behind config/flavor
```

---

## 4. Base layer — infrastructure primitives

Lives in `src/base/`. All modules here are framework-level; they don't know about AFFiNE business concepts.

### `ConfigModule` (`base/config/`)

Hierarchical typed configuration. Reads from environment variables, YAML files, and runtime overrides. Provides the `Config` injectable used everywhere. See `base/config/config.ts` for the full shape.

### `PrismaModule` (`base/prisma/`)

Wraps `PrismaClient` and registers it in the NestJS DI container. Supports `@Transactional()` via `nestjs-cls/transactional`.

```ts
// Usage in any service
constructor(private readonly models: Models) {}
// models is a facade over PrismaClient — see Models layer below
```

### `RedisModule` (`base/redis/`)

Provides an `ioredis` client. Used by cache, BullMQ job queues, Socket.io adapter (pub/sub for multi-instance sync), and distributed mutex.

### `CacheModule` (`base/cache/`)

Simple typed in-memory + Redis cache layer. Used for rate limiting, session lookups, and config caching.

### `MutexModule` (`base/mutex/`)

Redis-backed distributed mutex (`Mutex` injectable). Used in `DocStorageAdapter` to prevent concurrent snapshot merges for the same doc.

### `EventModule` (`base/event/`)

Internal typed event bus (`EventBus`). Uses `eventemitter2`. Modules declare their events by augmenting a global `Events` interface:

```ts
// In any module's .ts file
declare global {
  interface Events {
    'doc.snapshot.updated': { workspaceId: string; docId: string; blob: Buffer };
  }
}

// Publishing
this.event.emit('doc.snapshot.updated', { ... });

// Subscribing (via decorator in a NestJS service)
@OnEvent('doc.snapshot.updated')
async handleSnapshotUpdate(payload: Events['doc.snapshot.updated']) { ... }
```

### `JobModule` (`base/job/`)

BullMQ-backed async job queues. Modules declare their jobs by augmenting a global `Jobs` interface (same pattern as events). Provides `JobQueue` (for enqueuing) and `@OnJob()` decorator (for processing).

```ts
declare global {
  interface Jobs {
    'copilot.session.generateTitle': { sessionId: string };
  }
}

// Enqueue
await this.queue.add('copilot.session.generateTitle', { sessionId });

// Process
@OnJob('copilot.session.generateTitle')
async handleGenerateTitle(job: Job<Jobs['copilot.session.generateTitle']>) { ... }
```

### `GqlModule` (`base/graphql/`)

Configures `@nestjs/apollo` with code-first schema generation. Installed only when `env.flavors.graphql` is true.

### `MetricsModule` (`base/metrics/`)

OpenTelemetry metrics + traces. Exports Prometheus metrics, Zipkin traces, and Google Cloud Trace (when running on GCP). The `metrics` object provides counters/histograms used throughout.

### `WebSocketModule` (`base/websocket/`)

`SocketIoAdapter` — replaces NestJS's default WS adapter with Socket.io backed by a Redis adapter, enabling horizontal scaling across multiple sync server instances.

### `StorageProviderModule` (`base/storage/`)

Blob storage providers: local filesystem or S3-compatible. The `StorageProvider` injectable is used by `BlobStorage` in the core layer.

### `RateLimiterModule` (`base/throttler/`)

`CloudThrottlerGuard` — rate-limits by user ID (authenticated) or IP (anonymous). Configured per-route.

### `LoggerModule` (`base/logger/`)

Winston-based structured logger (`AFFiNELogger`). Replaces NestJS's built-in logger.

---

## 5. Models layer — database access

`src/models/` contains one TypeScript class per major entity. Each class is an `@Injectable()` that receives `PrismaClient` and exposes typed methods — no raw SQL leaks into business logic.

Key models:

| File                 | Entity                             | Notable methods                                            |
| -------------------- | ---------------------------------- | ---------------------------------------------------------- |
| `user.ts`            | `User`                             | `create`, `findByEmail`, `update`, `delete`                |
| `workspace.ts`       | `Workspace`                        | `create`, `findById`, `listByUser`                         |
| `workspace-user.ts`  | `WorkspaceUserRole`                | `setRole`, `getRoleByUser`, `list`                         |
| `doc.ts`             | `Snapshot` + `Update`              | `upsertSnapshot`, `pushUpdate`, `getUpdates`, `getHistory` |
| `blob.ts`            | `Blob`                             | `upsert`, `findByKey`, `delete`, `listByWorkspace`         |
| `session.ts`         | `Session` + `UserSession`          | `createSession`, `getSessionById`, `refreshSession`        |
| `copilot-session.ts` | `AiSession`                        | `create`, `update`, `list`, `fork`                         |
| `notification.ts`    | `Notification`                     | `create`, `list`, `markRead`                               |
| `feature.ts`         | `UserFeature` + `WorkspaceFeature` | `enable`, `disable`, `has`                                 |

All models are re-exported from `src/models/index.ts` as a single `Models` aggregate:

```ts
// Inject in any service
constructor(private readonly models: Models) {}

// Use
const user = await this.models.user.findByEmail('user@example.com');
const docs = await this.models.doc.getUpdates(workspaceId, docId);
```

---

## 6. Core modules — business logic

### `AuthModule` (`core/auth/`)

Handles sessions, cookies, and request authentication.

- **`AuthService`** — creates/validates sessions. Session token stored in `affine_session` cookie (HttpOnly). Also handles CSRF via `affine_csrf_token`.
- **`AuthGuard`** — global NestJS guard. Reads cookie or `Authorization: Bearer <token>` header. Populates `CurrentUser` on the request context via `@CurrentUser()` decorator.
- **`AuthResolver`** (GraphQL) — `signIn`, `signOut`, `signUp`, `sendEmailVerification` mutations.
- **`AuthController`** (REST) — `/auth/sign-in`, `/auth/sign-out` endpoints used by non-GraphQL clients (Electron, mobile).

Session lookup is cached in Redis to avoid a DB hit on every request.

### `UserModule` (`core/user/`)

CRUD for `User` entity. `UserResolver` exposes `me`, `user`, `updateUser`, `deleteUser` GraphQL operations. `UserService` wraps `Models.user` with business rules (e.g., cannot delete workspace owner).

### `WorkspaceModule` (`core/workspaces/`)

Workspace creation, member management, invitation flow. Key operations:

- `createWorkspace` — creates workspace + sets creator as Owner
- `inviteUser` — creates `WorkspaceUserRole` with `Pending` status, sends email
- `acceptInvite` — transitions status to `Accepted`
- `leaveWorkspace` / `kickMember` — removes role entry

### `PermissionModule` (`core/permission/`)

`AccessController` — the central authorization service. Used across all modules that gate on workspace/doc access.

```ts
// Check if user can read a doc
await this.permission.checkDocAccess(userId, workspaceId, docId, 'Read');

// Check workspace access
await this.permission.checkWorkspaceAccess(userId, workspaceId, WorkspaceAction.Read);
```

Throws typed errors (`SpaceAccessDenied`, `DocNotFound`) rather than returning booleans, so they propagate cleanly through the GraphQL/WS error handlers.

### `DocStorageModule` (`core/doc/`)

The persistence layer for Yjs documents. AFFiNE stores documents as Yjs CRDT update binaries.

**Two adapters, same interface (`DocStorageAdapter`):**

| Adapter                        | Table                             | Purpose                          |
| ------------------------------ | --------------------------------- | -------------------------------- |
| `PgWorkspaceDocStorageAdapter` | `snapshots` + `updates`           | Workspace docs (pages, edgeless) |
| `PgUserspaceDocStorageAdapter` | `user_snapshots` + `user_updates` | Per-user private docs            |

**Write path:**

1. `pushDocUpdates(workspaceId, docId, updates[])` — inserts raw Yjs update binaries into the `updates` table.
2. A background job (`doc:merge-updates`) periodically merges pending updates into the `snapshot` via `mergeUpdatesInApplyWay()` (calling into `@affine/server-native` Rust code).
3. After merge, fires `doc.snapshot.updated` event which triggers indexing/embedding.

**Read path:**

1. `getDoc(workspaceId, docId)` — fetches the latest `snapshot` + any pending `updates` not yet merged.
2. Client applies these on top of its local CRDT state.

**History:**
Snapshots before merge are archived to `snapshot_history` with a retention policy. `getDocHistory` / `rollbackDoc` operate on this table.

### `SyncModule` (`core/sync/`)

Real-time WebSocket gateway. Uses Socket.io rooms.

```
Client connects → authenticates via cookie/token
  → joins a "space" room (workspace or userspace)
  → sends doc updates → server writes to DB + broadcasts to room
  → receives updates from others via room broadcast
```

Room naming: `${spaceId}:sync-026` (current protocol), `${spaceId}:awareness:${docId}`.

Client version is checked on connect — clients older than `0.25.0` are rejected. Protocol version `0.26.0+` uses batched update events (`space:broadcast-doc-updates`).

**Key Socket.io events:**

| Event                         | Direction     | Description                                 |
| ----------------------------- | ------------- | ------------------------------------------- |
| `space:join`                  | client→server | Join a workspace/userspace room             |
| `space:leave`                 | client→server | Leave room                                  |
| `space:push-doc-updates`      | client→server | Push Yjs updates                            |
| `space:load-doc`              | client→server | Request full doc state (snapshot + updates) |
| `space:broadcast-doc-updates` | server→client | Broadcast new updates to room members       |
| `space:join-awareness`        | client→server | Subscribe to cursor/presence for a doc      |
| `awareness:update`            | bidirectional | Cursor/presence state                       |

### `NotificationModule` (`core/notification/`)

In-app notifications (workspace invites, mentions, doc shares). Stored in `notifications` table. Delivered via WebSocket push to connected clients when they join.

### `QuotaModule` (`core/quota/`)

Enforces storage and feature limits per user/workspace. Quotas are attached via `UserFeature` / `WorkspaceFeature` records. `QuotaService.checkBlobQuota()` gates blob uploads; `checkStorageQuota()` gates snapshot writes.

### `FeatureModule` (`core/features/`)

Feature flags at user and workspace level. Features are named strings stored in `user_features` / `workspace_features`. Used to gate early access, beta features, and plan-specific capabilities.

### `StorageModule` (`core/storage/`)

Blob storage for attachments (images, PDFs, etc.). `StorageResolver` exposes `createCheckout`-style GraphQL mutations and REST endpoints for direct upload/download. Routes blobs through `StorageProviderModule` (S3/local).

### `MailModule` (`core/mail/`)

Sends transactional emails via Nodemailer. Email templates are React components in `src/mails/` rendered with `@react-email/components`. `Mailer` service used by auth (verification, magic link) and workspace modules (invitations).

### `DocRendererModule` (`core/doc-renderer/`)

Renders Yjs docs to HTML for link previews and SSR. Loaded in `renderer` and `front` flavors.

### `DocServiceModule` (`core/doc-service/`)

HTTP endpoints for doc CRUD, used internally between microservice flavors. Loaded in `doc` and `front` flavors.

---

## 7. Plugin modules — optional features

### `CopilotModule` (`plugins/copilot/`)

AI assistant. See [Section 9](#9-ai-copilot-deep-dive) for detail.

### `PaymentModule` (`plugins/payment/`)

Stripe subscriptions and RevenueCAT (mobile). Handles:

- Subscription creation, upgrade, downgrade, cancellation via Stripe Checkout
- Webhook processing (`stripe.ts`) — activates/deactivates features based on subscription status
- License management for self-hosted Pro (`plugins/license/`)
- `SubscriptionService` is depended on by `CopilotModule` to check if a user has AI quota

### `OAuthModule` (`plugins/oauth/`)

OAuth2 provider integrations (Google, GitHub, etc.). `OAuthController` handles the redirect dance. On success, creates/links a `ConnectedAccount` and signs in the user.

### `IndexerModule` (`plugins/indexer/`)

Full-text search and AI embedding indexing. Listens to `doc.snapshot.updated` events and queues index jobs. Uses pgvector for semantic search.

### `CalendarModule` (`plugins/calendar/`)

Google Calendar integration. `CalendarAccount` stores OAuth tokens; `CalendarEvent` caches upcoming events for the AI copilot's context.

### `WorkerModule` (`plugins/worker/`)

Background job processor. Loaded only on `allinone` / self-hosted. Processes BullMQ jobs from all modules (doc merge, embedding, email, etc.).

### `CaptchaModule` (`plugins/captcha/`)

hCaptcha / Cloudflare Turnstile verification for sign-up and sensitive operations.

---

## 8. Document sync deep-dive

### The update pipeline

```
Client edits doc
  │
  ▼
[Socket.io] space:push-doc-updates
  │  (array of Uint8Array Yjs update binaries)
  │
  ▼
SyncGateway.pushDocUpdates()
  │
  ├─→ PgWorkspaceDocStorageAdapter.pushDocUpdates()
  │     └─ INSERT INTO updates (workspace_id, doc_id, blob, created_by)
  │
  └─→ broadcast to room  (space:broadcast-doc-updates)
        └─ all other clients apply the update locally
```

### Snapshot merge (background)

```
JobQueue: 'doc:merge-updates'
  │
  ▼
DocStorageCronJob (periodic) or event-triggered
  │
  ▼
DocStorageAdapter.mergeUpdates()
  │
  ├─ SELECT all updates for doc WHERE merged = false
  ├─ native.mergeUpdatesInApplyWay(updates[])  ← Rust/NAPI
  │    └─ applies all updates to a fresh y-octo Doc, returns encoded snapshot
  ├─ UPSERT snapshots SET blob = merged_blob
  ├─ UPDATE updates SET merged = true
  └─ emit 'doc.snapshot.updated'
```

### Why Rust for merge?

Merging Yjs updates in JavaScript is slow for large docs. `@affine/server-native` implements merge using `y-octo` (Rust Yjs) which is 10–20× faster and avoids V8 GC pressure.

---

## 9. AI Copilot deep-dive

The copilot module (`plugins/copilot/`) is a multi-provider, session-based AI chat system.

### Components

```
CopilotModule
  ├── CopilotProviderFactory   — picks which AI provider to use for a request
  ├── PromptService            — manages system prompts stored in DB
  ├── CopilotSessionService    — creates/manages chat sessions (AiSession in DB)
  ├── CopilotResolver          — GraphQL entry points
  ├── WorkflowService          — graph-based multi-step AI workflows
  └── McpService               — Model Context Protocol tool integration
```

### Provider system

Each AI provider implements a common `CopilotProvider` interface. The factory selects a provider based on `modelId`, configured priorities, and feature flags.

| Provider file   | Backend                                                        |
| --------------- | -------------------------------------------------------------- |
| `openai.ts`     | OpenAI Chat Completions / Responses API                        |
| `anthropic/`    | Anthropic Claude (via `@affine/server-native` LLM dispatcher)  |
| `gemini/`       | Google Gemini (via native LLM dispatcher)                      |
| `fal.ts`        | Fal.ai (image generation)                                      |
| `cloudflare.ts` | Cloudflare Workers AI                                          |
| `perplexity.ts` | Perplexity (web search)                                        |
| `morph.ts`      | Morph (code editing)                                           |
| `native.ts`     | Routes through `@affine/server-native` for supported providers |

The `native.ts` provider calls Rust functions from `@affine/server-native` (`llm_dispatch`, `llm_dispatch_stream`, `llm_embedding_dispatch`) for providers supported by the native `llm_adapter` crate. This offloads HTTP I/O and middleware processing to Rust threads, keeping Node.js event loop free.

### Session lifecycle

```
createCopilotSession(workspaceId, promptName, docId?)
  └─ AiSession row created in DB

message sent (text / image / attachments)
  ├─ quota checked  (SubscriptionService + QuotaService)
  ├─ prompt assembled  (system prompt + history + user message)
  ├─ provider selected  (CopilotProviderFactory)
  └─ response streamed back via GraphQL subscription or REST SSE

session persisted (messages stored in AiSession.messages JSONB)
```

### Workflow system (`plugins/copilot/workflow/`)

Graph-based pipeline for multi-step operations (e.g., "summarize doc", "rename based on content"). Each workflow is a directed graph of nodes; nodes call providers, transform data, or branch conditionally.

---

## 10. Authentication flow

### Cookie-based (web)

```
POST /auth/sign-in  (or GraphQL signIn mutation)
  │
  ▼
AuthService.signIn(email, password)
  ├─ hash compare with stored argon2 hash
  ├─ create Session + UserSession records
  └─ set affine_session=<sessionId>; affine_user_id=<userId> cookies (HttpOnly)

Subsequent requests:
  AuthGuard.canActivate()
    ├─ reads affine_session cookie
    ├─ validates via Models.session.getUserBySession()  (Redis-cached)
    └─ populates CurrentUser on request context
```

### Token-based (Electron / mobile / API)

```
POST /auth/sign-in  → returns { token: "<jwt>" }

Subsequent requests:
  Authorization: Bearer <token>
  AuthGuard reads token, validates JWT, resolves user
```

### OAuth (Google, GitHub, …)

```
GET /oauth/login?provider=google  → redirect to provider
GET /oauth/callback?code=...
  ├─ exchange code for access token
  ├─ fetch user profile
  ├─ upsert User + ConnectedAccount
  └─ set session cookies  (same as password auth)
```

### Magic link / OTP

```
POST /auth/send-magic-link  → email sent with link
GET /auth/magic-link?token=...
  ├─ validate OTP record
  └─ create session  (same as password auth)
```

---

## 11. Database schema overview

PostgreSQL with `pgvector` extension. Managed by Prisma (`schema.prisma`).

```
users
  ├── user_sessions          (active login sessions)
  ├── user_features          (feature flags / plan)
  ├── user_connected_accounts (OAuth provider links)
  ├── user_settings
  └── access_tokens          (long-lived API tokens)

workspaces
  ├── workspace_user_permissions  (roles: Owner/Admin/Collaborator)
  ├── workspace_page_user_permissions  (per-doc role overrides)
  ├── workspace_features       (workspace-level feature flags)
  ├── workspace_pages          (doc metadata: title, mode, public flag)
  ├── blobs                    (binary attachments)
  ├── snapshots                (latest merged Yjs doc state)
  ├── updates                  (pending Yjs update queue)
  ├── snapshot_history         (archived snapshots for rollback)
  └── workspace_admin_stats    (aggregate stats for admin panel)

ai_sessions                    (copilot chat sessions)
  └── messages stored in JSONB field

ai_workspace_files             (files indexed for copilot context)
ai_embeddings                  (pgvector embeddings for semantic search)

stripe_customers / subscriptions / prices
calendar_accounts / events
notifications
comments / replies / comment_attachments
```

---

## 12. Config system

The `Config` injectable (`base/config/config.ts`) is the single source of truth for runtime configuration. Shape is defined via typed classes with decorators. Values come from:

1. Environment variables (highest priority)
2. YAML config files in `~/.affine/config/`
3. Code defaults

```ts
// Inject in any service
constructor(private readonly config: Config) {}

// Usage examples
this.config.server.port          // HTTP port
this.config.server.externalUrl   // public-facing URL
this.config.auth.session.ttl     // session expiry
this.config.copilot.providers    // AI provider configs
this.config.storage.r2.bucket    // blob storage bucket
```

---

## 13. Background jobs

BullMQ queues backed by Redis. Job types are declared by augmenting the global `Jobs` interface so everything is typed end-to-end.

Key jobs and who processes them:

| Job name                        | Enqueued by                    | Processed by            |
| ------------------------------- | ------------------------------ | ----------------------- |
| `doc:merge-updates`             | `DocStorageCronJob` (periodic) | `DocStorageAdapter`     |
| `doc:delete`                    | Workspace delete flow          | `DocStorageAdapter`     |
| `copilot.session.generateTitle` | After first AI response        | `CopilotSessionService` |
| `copilot.session.deleteDoc`     | Doc delete event               | `CopilotSessionService` |
| `indexer:index-workspace`       | Workspace created/updated      | `IndexerModule`         |
| `indexer:embed-doc`             | `doc.snapshot.updated` event   | `IndexerModule`         |
| `mail:send`                     | Various auth/workspace flows   | `MailModule`            |

In `allinone` / self-hosted, `WorkerModule` processes all queues in the same process. In cloud, dedicated `worker` flavor containers pick up jobs.

---

## 14. Key patterns

### Typed global event augmentation

```ts
// Any module can add events without modifying a central file
declare global {
  interface Events {
    'my-module.something.happened': { id: string };
  }
}
```

Same pattern for `Jobs` and `Metrics`.

### Error handling

All business errors extend a base class in `@affine/error`. `GlobalExceptionFilter` catches them and formats them as GraphQL errors (with `extensions.code`) or HTTP responses. Never throw plain `Error` in business logic — use the typed errors:

```ts
import { SpaceAccessDenied, DocNotFound } from '../../base';
throw new SpaceAccessDenied();
```

### Transactional operations

Use `@Transactional()` from `@nestjs-cls/transactional` for multi-step DB writes:

```ts
@Transactional()
async createWorkspaceWithOwner(userId: string) {
  const workspace = await this.models.workspace.create();
  await this.models.workspaceUser.setRole(workspace.id, userId, 'Owner');
  // Both writes share the same Prisma transaction
}
```

### Request-scoped context (CLS)

`nestjs-cls` provides a `ClsService` scoped to the current HTTP request or WebSocket message. Carries the request ID (for tracing) and resolved hostname. Access via `CLS_REQUEST_HOST` key.

### Metrics instrumentation

```ts
// Wrapping a method with automatic duration histogram
@CallMetric('socketio', 'event_duration', { event: 'push-doc-updates' })
async pushDocUpdates(...) { ... }
```
