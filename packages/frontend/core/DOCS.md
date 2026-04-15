# @affine/core — Code Documentation

Shared application logic used by every AFFiNE frontend (web, Electron, iOS, Android). 69+ modules wired together via `@toeverything/infra` DI. No module imports from another module's internals — everything goes through the DI container.

---

## Table of Contents

1. [Directory layout](#1-directory-layout)
2. [Module anatomy](#2-module-anatomy)
3. [Scope hierarchy](#3-scope-hierarchy)
4. [All modules — DI registrations](#4-all-modules--di-registrations)
5. [Core modules in depth](#5-core-modules-in-depth)
6. [Cloud module — full breakdown](#6-cloud-module--full-breakdown)
7. [Workbench module — browser vs desktop](#7-workbench-module--browser-vs-desktop)
8. [BlockSuite integration](#8-blocksuite-integration)
9. [Shared components](#9-shared-components)
10. [Patterns reference](#10-patterns-reference)
11. [How apps wire core](#11-how-apps-wire-core)

---

## 1. Directory layout

```
packages/frontend/core/src/
  modules/            ← 69 domain modules (each has index.ts + services/entities/stores/scopes)
  blocksuite/         ← BlockSuite view-extensions + editor setup (NOT React)
  components/         ← Shared React UI not in @affine/component
  desktop/            ← Desktop-only router/layout wiring
  mobile/             ← Mobile-only router/layout wiring
  pages/              ← Route-level page components
  utils/              ← App-level utilities
  index.ts            ← Re-exports all configureXxxModule() functions
```

---

## 2. Module anatomy

Every module follows the same layout:

```
modules/example/
  index.ts            ← configureExampleModule(framework) + public re-exports
  services/
    example.ts        ← ExampleService extends Service
  entities/
    example.ts        ← Example extends Entity<PropsType>
  stores/
    example.ts        ← ExampleStore extends Store
  scopes/
    example.ts        ← ExampleScope extends Scope<PropsType>
  providers/
    example.ts        ← ExampleProvider (abstract, for .impl() swapping)
  impls/
    browser.ts        ← BrowserExampleImpl
    desktop.ts        ← DesktopExampleImpl
```

**DI registration pattern:**

```ts
// modules/doc/index.ts
export function configureDocModule(framework: Framework) {
  framework
    .scope(WorkspaceScope) // register inside WorkspaceScope
    .service(DocsService, [
      // service + its dep list
      DocsStore,
      DocPropertiesStore,
      [DocCreateMiddleware], // [] = optional multi-provider
    ])
    .store(DocPropertiesStore, [WorkspaceService, WorkspaceDBService])
    .store(DocsStore, [WorkspaceService, DocPropertiesStore])
    .entity(DocRecord, [DocsStore, DocPropertiesStore])
    .entity(DocRecordList, [DocsStore])
    .scope(DocScope)
    .entity(Doc, [DocScope, DocsStore, WorkspaceService])
    .service(DocService);
}
```

**Consuming in React:**

```ts
const docsService = useService(DocsService);
const docs = useLiveData(docsService.docs$);

// Safe — returns null if service not registered
const desktopApi = useServiceOptional(DesktopApiService);
```

---

## 3. Scope hierarchy

Scopes control service lifetime. A scope is destroyed when its parent scope is destroyed or when you explicitly close it.

```
[Global Scope]   — lives for the entire app session
  LifecycleService
  FeatureFlagService
  GlobalStateService          (localStorage / electron-store)
  GlobalCacheService          (localStorage)
  GlobalSessionStateService   (sessionStorage)
  NbstoreService              (nbstore worker host)
  AppThemeService
  WorkspacesService
  ServersService
  DefaultServerService

  [ServerScope]   — one per AFFiNE server (affine.pro + self-hosted)
    ServerService
    FetchService
    GraphQLService
    AuthService
    AuthSession               (entity)
    SubscriptionService       Subscription, SubscriptionPrices (entities)
    UserQuotaService          UserQuota (entity)
    UserFeatureService        UserFeature (entity)
    UserCopilotQuotaService   UserCopilotQuota (entity)
    InvoicesService           Invoices (entity)
    NotificationService, NotificationCountService, NotificationListService
    EventSourceService
    CaptchaService
    PublicUserService
    UserSettingsService
    AccessTokenService

  [WorkspaceScope]   — one per open workspace
    WorkspaceService
    Workspace                 (entity: id, name$, avatar$, rootYDoc, docCollection, engine)
    WorkspaceEngine           (entity: doc/blob/indexer/awareness frontends)
    WorkspaceDBService        (Yjs ORM: db.tags, db.favorites, db.docProperties, ...)
    DocsService               DocRecord, DocRecordList (entities)
    WorkbenchService          Workbench (entity: views$, activeView$, sidebarOpen$)
    TagService                TagList, Tag (entities)
    CollectionService         Collection (entity)
    FavoriteService           FavoriteList (entity)
    OrganizeService           FolderTree, FolderNode (entities)
    PermissionsService        WorkspacePermission, WorkspaceMembers (entities)
    QuickSearchService        QuickSearch (entity)
    DocsSearchService
    JournalService
    WorkspaceServerService
    WorkspaceSubscriptionService  WorkspaceSubscription (entity)
    WorkspaceInvoicesService      WorkspaceInvoices (entity)
    SelfhostLicenseService
    NavigatorService          Navigator (entity)
    ...

    [DocScope]   — one per open document
      DocService
      Doc                     (entity: id, title$, primaryMode$, trash$, yDoc, blockSuiteDoc)
      EditorsService
      Editor                  (entity: mode$, selector$, editorContainer$)
      JournalDocService
      DocUpdatedByService
      CloudDocMetaService     CloudDocMeta (entity)
      ShareInfoService        ShareInfo (entity)
      CommentService

      [EditorScope]   — one per Editor instance
        EditorService

    [ViewScope]   — one per workbench tab/view
      ViewService
```

---

## 4. All modules — DI registrations

### `storage` module

```ts
export function configureStorageModule(framework: Framework) {
  framework.service(GlobalStateService, [GlobalState]).service(GlobalCacheService, [GlobalCache]).service(GlobalSessionStateService, [GlobalSessionState]).service(NbstoreService, [NbstoreProvider]);
}

// Per-platform impls:
export function configureLocalStorageStateStorageImpls(framework: Framework) {
  framework.impl(GlobalCache, LocalStorageGlobalCache).impl(GlobalState, LocalStorageGlobalState).impl(CacheStorage, IDBGlobalState);
}
// Electron replaces with electron-store backed impls
// Mobile apps use same localStorage impls
```

**Key providers:**

- `GlobalState` — persistent key-value (survives app restarts)
- `GlobalCache` — semi-persistent (may be evicted)
- `GlobalSessionState` — session-only (cleared on tab close)
- `NbstoreProvider` — abstract factory for opening nbstore workspace stores

---

### `lifecycle` module

```ts
export function configureLifecycleModule(framework: Framework) {
  framework.service(LifecycleService);
}
```

```ts
class LifecycleService extends Service {
  applicationStart(): void; // emits ApplicationStarted event
  applicationFocus(): void; // emits ApplicationFocused event
}

// Events (for @OnEvent decorators in other services):
const ApplicationStarted = createEvent<boolean>('ApplicationStartup');
const ApplicationFocused = createEvent<boolean>('ApplicationFocused');
```

App entry points call `lifecycleService.applicationStart()` once on boot and `applicationFocus()` on window/tab focus.

---

### `feature-flag` module

```ts
export function configureFeatureFlagModule(framework: Framework) {
  framework.service(FeatureFlagService).entity(Flags, [GlobalStateService]);
}
```

```ts
class FeatureFlagService extends Service {
  readonly flags: Flags; // typed access to all AFFINE_FLAGS
}

// Usage:
const enabled = featureFlagService.flags.enable_battery_save_mode.value;
// or reactively:
const enabled$ = featureFlagService.flags.enable_battery_save_mode.$;
```

Flags are defined in `modules/feature-flag/constant.ts` as `AFFINE_FLAGS`. Each flag has a `configurable` boolean — non-configurable flags are hard-coded.

---

### `workspace` module

```ts
export function configureWorkspaceModule(framework: Framework) {
  framework
    .service(WorkspacesService, [WorkspaceFlavoursService, WorkspaceListService, WorkspaceProfileService, WorkspaceTransformService, WorkspaceRepositoryService, WorkspaceFactoryService, WorkspaceDestroyService])
    .service(WorkspaceFlavoursService, [[WorkspaceFlavoursProvider]])
    .service(WorkspaceDestroyService, [WorkspaceFlavoursService])
    .service(WorkspaceListService)
    .entity(WorkspaceList, [WorkspaceFlavoursService])
    .service(WorkspaceProfileService)
    .store(WorkspaceProfileCacheStore, [GlobalCache])
    .entity(WorkspaceProfile, [WorkspaceProfileCacheStore, WorkspaceFlavoursService])
    .service(WorkspaceFactoryService, [WorkspaceFlavoursService])
    .service(WorkspaceTransformService, [WorkspaceFactoryService, WorkspaceDestroyService])
    .service(WorkspaceRepositoryService, [WorkspaceFlavoursService, WorkspaceProfileService, WorkspaceListService])
    .scope(WorkspaceScope)
    .service(WorkspaceService)
    .entity(Workspace, [WorkspaceScope, FeatureFlagService])
    .service(WorkspaceEngineService, [WorkspaceScope])
    .entity(WorkspaceEngine, [WorkspaceService, NbstoreService, FeatureFlagService]);
  // ... more workspace-scoped services
}
```

**`WorkspacesService`** — global singleton, delegates to sub-services:

```ts
class WorkspacesService extends Service {
  get list(): WorkspaceList; // entity: workspaces$.value: WorkspaceMetadata[]
  open(meta: WorkspaceMetadata): WorkspaceScope;
  openByWorkspaceId(id: string): WorkspaceScope | null;
  create(flavour: string, initial: InitialDocPage): Promise<WorkspaceMetadata>;
  deleteWorkspace(meta: WorkspaceMetadata): Promise<void>;
  getProfile(meta: WorkspaceMetadata): WorkspaceProfile;
  transformLocalToCloud(meta: WorkspaceMetadata): Promise<void>;
  getAllWorkspaceProfile(): WorkspaceProfile[];
  getWorkspaceBlob(meta: WorkspaceMetadata, blob: string): Promise<Blob | null>;
}
```

**`Workspace`** entity — root entity of a workspace scope:

```ts
class Workspace extends Entity {
  readonly id: string;
  readonly flavour: 'local' | 'affine-cloud';
  readonly meta: WorkspaceMetadata;
  readonly openOptions: WorkspaceOpenOptions;
  readonly rootYDoc: YDoc; // root Yjs document (guid = workspace id)

  // Reactive fields backed by Yjs map:
  name$: LiveData<string | undefined>;
  avatar$: LiveData<string | undefined>;

  // Lazy-created BlockSuite doc collection:
  get docCollection(): WorkspaceInterface;

  // Engine for doc/blob/indexer/awareness ops:
  get engine(): WorkspaceEngine;

  get docs(): DocsService;
  get canGracefulStop(): boolean;
}
```

**`WorkspaceEngine`** entity:

```ts
class WorkspaceEngine extends Entity {
  client?: StoreClient; // nbstore worker client
  started: boolean;

  get doc(): DocFrontend; // Yjs doc storage frontend
  get blob(): BlobFrontend; // Binary attachment frontend
  get indexer(): IndexerFrontend; // Full-text search frontend
  get awareness(): AwarenessFrontend; // Real-time presence frontend

  start(): void; // Opens nbstore, connects docs to engine
  stop(): void;
}
```

**Workspace flavours** are pluggable. Each flavour registers via `WorkspaceFlavoursProvider`:

```ts
// Browser (web app):
framework.impl(WorkspaceFlavoursProvider('LOCAL'), LocalWorkspaceFlavoursProvider).impl(WorkspaceFlavoursProvider('CLOUD'), CloudWorkspaceFlavoursProvider, [GlobalState, ServersService]);

// Electron adds its own SQLite-backed local flavour
```

---

### `doc` module

```ts
export function configureDocModule(framework: Framework) {
  framework
    .scope(WorkspaceScope)
    .service(DocsService, [DocsStore, DocPropertiesStore, [DocCreateMiddleware]])
    .store(DocPropertiesStore, [WorkspaceService, WorkspaceDBService])
    .store(DocsStore, [WorkspaceService, DocPropertiesStore])
    .entity(DocRecord, [DocsStore, DocPropertiesStore])
    .entity(DocRecordList, [DocsStore])
    .scope(DocScope)
    .entity(Doc, [DocScope, DocsStore, WorkspaceService])
    .service(DocService);
}
```

**`DocsService`** — manages all docs in a workspace:

```ts
class DocsService extends Service {
  createDoc(options?: { id?: string; title?: string; mode?: DocMode; skipInit?: boolean; primaryMode?: DocMode }): DocRecord;

  deleteDoc(docId: string): void;
  duplicateDoc(docId: string): DocRecord;
  list: DocRecordList; // entity with docs$ LiveData
}
```

**`DocRecord`** entity — lightweight metadata record (no Yjs doc loaded):

```ts
class DocRecord extends Entity {
  readonly id: string;
  meta$: LiveData<DocMeta>; // raw BlockSuite meta
  title$: LiveData<string>; // parsed title
  primaryMode$: LiveData<DocMode>; // 'page' | 'edgeless'
  trash$: LiveData<boolean>;
  trashDate$: LiveData<number | undefined>;
  createdAt$: LiveData<number | undefined>;
  updatedAt$: LiveData<number | undefined>;
  createdBy$: LiveData<string | undefined>; // user id
  updatedBy$: LiveData<string | undefined>; // user id

  setMeta(meta: Partial<DocMeta>): void;
  setMode(mode: DocMode): void;
  moveToTrash(): void;
  restoreFromTrash(): void;
  setProperty(propertyId: string, value: string): void;
  updateProperties(props: Partial<DocProperties>): void;
}
```

**`Doc`** entity — full document with Yjs doc loaded (exists within DocScope):

```ts
class Doc extends Entity {
  readonly id: string;
  readonly yDoc: YDoc; // the space-level Yjs doc
  readonly blockSuiteDoc: BlockSuiteDoc; // BlockSuite Document wrapper
  readonly record: DocRecord; // linked metadata record

  // Proxied from record:
  readonly meta$: LiveData<DocMeta>;
  readonly properties$: LiveData<DocProperties>;
  readonly primaryMode$: LiveData<DocMode>;
  readonly title$: LiveData<string>;
  readonly trash$: LiveData<boolean>;
  readonly createdAt$: LiveData<number | undefined>;
  readonly updatedAt$: LiveData<number | undefined>;
  readonly createdBy$: LiveData<string | undefined>;
  readonly updatedBy$: LiveData<string | undefined>;

  setCreatedAt(ts: number): void;
  setUpdatedAt(ts: number): void;
  setCreatedBy(userId: string): void;
  setUpdatedBy(userId: string): void;
  customProperty$(propertyId: string): LiveData<string | undefined>;
  setProperty(propertyId: string, value: string): void;
  updateProperties(props: Partial<DocProperties>): void;
  getProperties(): DocProperties;

  get workspace(): Workspace;
}
```

Doc's `afterTransaction` Yjs handler auto-updates `updatedAt` (throttled to 1s, leading+trailing). Doc also registers itself with the engine at priority 100 so it's synced first.

**`DocCreateMiddleware`** — optional multi-provider for post-create hooks:

```ts
// Register a middleware to run after every doc creation:
framework.impl(DocCreateMiddleware, MyMiddleware);

class MyMiddleware implements DocCreateMiddleware {
  afterCreate(doc: DocRecord): void { ... }
}
```

---

### `db` module

Yjs ORM — typed structured storage inside the workspace Yjs document.

```ts
export function configureWorkspaceDBModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(WorkspaceDBService, [WorkspaceService, WorkspaceServerService]).entity(WorkspaceDB).entity(WorkspaceDBTable, [WorkspaceService]);
}
```

```ts
class WorkspaceDBService extends Service {
  db: WorkspaceDB
}

// WorkspaceDB tables (all LiveData-based):
workspaceDB.tags.create({ id, value, color })
workspaceDB.tags.find$(id): Observable<Tag | null>
workspaceDB.tags.findAll$(): Observable<Tag[]>
workspaceDB.favorites.create({ id, index, ref, refId })
workspaceDB.favorites.findAll$(): Observable<FavoriteItem[]>
workspaceDB.docProperties.update(docId, { createDate, updatedDate, ... })
workspaceDB.docProperties.findAll$(): Observable<DocProperties[]>
workspaceDB.folders.create({ id, name, parentId, index })
workspaceDB.folders.findAll$(): Observable<FolderItem[]>
```

Schema types are defined in `modules/db/schema.ts`.

---

### `editor` module

```ts
export function configureEditorModule(framework: Framework) {
  framework.scope(WorkspaceScope).scope(DocScope).service(EditorsService).entity(Editor, [DocService, WorkspaceService]).scope(EditorScope).service(EditorService, [EditorScope]);
}
```

**`Editor`** entity — one per open editor instance:

```ts
class Editor extends Entity {
  readonly scope: EditorScope;
  readonly doc: Doc; // from DocService
  readonly isSharedMode: boolean;

  mode$: LiveData<DocMode>; // 'page' | 'edgeless'
  selector$: LiveData<EditorSelector | undefined>; // block/element selection
  editorContainer$: LiveData<AffineEditorContainer | null>;
  defaultOpenProperty$: LiveData<DefaultOpenProperty | undefined>;

  workbenchView: WorkbenchView | null;
  scrollPosition: {
    page: number | null;
    edgeless: { centerX: number; centerY: number; zoom: number } | null;
  };

  setMode(mode: DocMode): void;
  setSelector(selector: EditorSelector): void;
}
```

**`EditorService`** — service within EditorScope:

```ts
class EditorService extends Service {
  editor: Editor; // the Editor entity for this scope
}
```

**`EditorsService`** — manages multiple editor instances within a DocScope. Use it when you need more than one editor per doc.

---

### `tag` module

```ts
export function configureTagModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(TagService).store(TagStore, [WorkspaceService]).entity(TagList, [TagStore, DocsService]).entity(Tag, [TagStore, DocsService]);
}
```

```ts
class TagService extends Service {
  tagList: TagList; // entity with tags$ LiveData
  createTag(value: string, color: string): Tag;
  deleteTag(tagId: string): void;
  filterDocsByTag(tagId: string): DocRecord[];
}

class Tag extends Entity {
  readonly id: string;
  value$: LiveData<string>;
  color$: LiveData<string>;
  docs$: LiveData<DocRecord[]>; // all docs with this tag
}
```

---

### `collection` module

```ts
export function configureCollectionModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(CollectionService, [CollectionStore]).store(CollectionStore, [WorkspaceService]).entity(Collection, [CollectionStore, CollectionRulesService]).store(PinnedCollectionStore, [WorkspaceDBService]).service(PinnedCollectionService, [PinnedCollectionStore]);
}
```

```ts
class CollectionService extends Service {
  collections$: LiveData<CollectionMeta[]>;
  createCollection(options: CollectionInfo): Collection;
  deleteCollection(id: string): void;
  getCollection(id: string): Collection | null;
}

class Collection extends Entity {
  readonly id: string;
  name$: LiveData<string>;
  icon$: LiveData<string | undefined>;
  filterMode$: LiveData<'and' | 'or'>;
  filters$: LiveData<Filter[]>;
  allowList$: LiveData<string[]>; // doc ids explicitly added
  docs$: LiveData<DocRecord[]>; // evaluated doc list
}
```

---

### `organize` module

Manages workspace folder tree. Folders are stored in `WorkspaceDB`.

```ts
export function configureOrganizeModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(OrganizeService).entity(FolderTree, [FolderStore]).entity(FolderNode, [FolderStore]).store(FolderStore, [WorkspaceDBService]);
}
```

```ts
class OrganizeService extends Service {
  folderTree: FolderTree;
  createFolder(name: string, parentId?: string): FolderNode;
  moveNode(nodeId: string, newParentId: string, index: number): void;
}

class FolderNode extends Entity {
  readonly id: string;
  name$: LiveData<string>;
  parentId$: LiveData<string | null>;
  children$: LiveData<FolderNode[]>;
  type$: LiveData<'folder' | 'doc' | 'collection' | 'tag'>;
}
```

---

### `favorite` module

```ts
export function configureFavoriteModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(FavoriteService).entity(FavoriteList, [FavoriteStore]).store(FavoriteStore, [WorkspaceDBService]).service(MigrationFavoriteItemsAdapter, [WorkspaceService]).service(CompatibleFavoriteItemsAdapter, [FavoriteService]);
}
```

```ts
class FavoriteService extends Service {
  favoriteList: FavoriteList;
  toggleFavorite(type: FavoriteSupportType, id: string): void;
  isFavorite$(type: FavoriteSupportType, id: string): LiveData<boolean>;
}

// FavoriteSupportType: 'doc' | 'collection' | 'tag'
```

`MigrationFavoriteItemsAdapter` handles one-time data migration from the old favorites storage format. `CompatibleFavoriteItemsAdapter` provides backwards-compatible read access.

---

### `permissions` module

```ts
export function configurePermissionsModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(WorkspacePermissionService, [WorkspaceService, WorkspacesService, WorkspacePermissionStore]).store(WorkspacePermissionStore, [WorkspaceServerService, WorkspaceLocalState]).entity(WorkspacePermission, [WorkspaceService, WorkspacePermissionStore]).service(WorkspaceMembersService, [WorkspaceMembersStore]).entity(WorkspaceMembers, [WorkspaceService, WorkspaceMembersStore]).store(WorkspaceMembersStore, [WorkspaceServerService]).service(MemberSearchService, [MemberSearchStore]).store(MemberSearchStore, [WorkspaceServerService]).store(GuardStore, [WorkspaceServerService]).service(GuardService, [GuardStore]).scope(DocScope).service(DocGrantedUsersService, [DocGrantedUsersStore]).store(DocGrantedUsersStore, [WorkspaceServerService, DocService]);
}
```

```ts
class WorkspacePermissionService extends Service {
  permission: WorkspacePermission; // entity
  isOwner$: LiveData<boolean>;
  isAdmin$: LiveData<boolean>;
  isTeam$: LiveData<boolean>;
  isCollaborator$: LiveData<boolean>;
}

class GuardService extends Service {
  // Check if current user can perform an action:
  can(action: WorkspacePermissionActions): LiveData<boolean>;
  canDoc(action: DocPermissionActions): LiveData<boolean>;
}
```

---

### `quicksearch` module

Cmd+K search — implements multiple search sessions (docs, commands, tags, collections, links, recent docs, creation).

```ts
export function configureQuickSearchModule(framework: Framework) {
  framework
    .scope(WorkspaceScope)
      .service(QuickSearchService)
      .service(CMDKQuickSearchService, [QuickSearchService, WorkbenchService, DocsService])
      .service(RecentDocsService, [WorkspaceLocalState, DocsService])
      .entity(QuickSearch)
      .entity(CommandsQuickSearchSession, [GlobalContextService])
      .entity(DocsQuickSearchSession, [WorkspaceService, ...])
      .entity(TagsQuickSearchSession, [TagService])
      .entity(CollectionsQuickSearchSession, [CollectionService])
      .entity(RecentDocsQuickSearchSession, [RecentDocsService])
      .entity(CreationQuickSearchSession, [DocsService, WorkbenchService])
      .entity(LinksQuickSearchSession, [WorkspaceService])
      .entity(ExternalLinksQuickSearchSession)
      .entity(JournalsQuickSearchSession, [JournalService]);
}
```

```ts
class QuickSearchService extends Service {
  open(): void;
  close(): void;
  isOpen$: LiveData<boolean>;
  query$: LiveData<string>;
}

class CMDKQuickSearchService extends Service {
  // Aggregates all sessions, presents unified result list
  results$: LiveData<QuickSearchItem[]>;
}
```

---

### `navigation` module

```ts
export function configureNavigationModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(NavigatorService).entity(Navigator, [WorkbenchService]);
}
```

```ts
class Navigator extends Entity {
  // Browser-history-style navigation within the workbench:
  canGoBack$: LiveData<boolean>;
  canGoForward$: LiveData<boolean>;
  goBack(): void;
  goForward(): void;
}
```

Helper functions from `modules/navigation/utils`:

```ts
resolveLinkToDoc(url: string): { docId: string; params: ReferenceParams } | null
resolveRouteLinkMeta(to: string): RouteLinkMeta | null
toDocSearchParams(params: ReferenceParams): URLSearchParams
toURLSearchParams(params: object): URLSearchParams
```

---

### `dialogs` module

Imperative modal system — open modals from services without React prop drilling.

```ts
export function configureDialogModule(framework: Framework) {
  framework.service(GlobalDialogService).scope(WorkspaceScope).service(WorkspaceDialogService);
}
```

```ts
class GlobalDialogService extends Service {
  open(key: keyof GLOBAL_DIALOG_SCHEMA, props?: object): () => void;
  // key examples: 'sign-in', 'deleted-account', 'setting'
}

class WorkspaceDialogService extends Service {
  open(key: keyof WORKSPACE_DIALOG_SCHEMA, props?: object): () => void;
  // key examples: 'create-doc', 'import', 'collection-editor',
  //               'doc-info', 'move-to-trash'
}
```

Dialogs are registered in `modules/dialogs/constant.ts` as typed schemas.

---

### `theme` module

```ts
export function configureAppThemeModule(framework: Framework) {
  framework.service(AppThemeService).entity(AppTheme);
}
```

```ts
class AppThemeService extends Service {
  theme: AppTheme;
}

class AppTheme extends Entity {
  theme$: LiveData<'light' | 'dark' | 'system'>;
  resolvedTheme$: LiveData<'light' | 'dark'>; // resolved from OS if 'system'

  setTheme(theme: 'light' | 'dark' | 'system'): void;
}
```

---

### `notification` module

Server-side notifications (bell icon in header). Requires being in a `ServerScope`.

```ts
export function configureNotificationModule(framework: Framework) {
  framework.scope(ServerScope).service(NotificationService, [NotificationStore]).service(NotificationCountService, [NotificationStore, AuthService]).service(NotificationListService, [NotificationStore, NotificationCountService]).store(NotificationStore, [GraphQLService, ServerService, GlobalSessionState]);
}
```

```ts
class NotificationService extends Service {
  markAllAsRead(): Promise<void>;
  markAsRead(notificationId: string): Promise<void>;
}

class NotificationCountService extends Service {
  unreadCount$: LiveData<number>;
}

class NotificationListService extends Service {
  notifications$: LiveData<Notification[]>;
  hasMore$: LiveData<boolean>;
  loadMore(): Promise<void>;
}
```

---

### `docs-search` module

Full-text search across the workspace using the nbstore indexer.

```ts
export function configureDocsSearchModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(DocsSearchService, [WorkspaceService, DocsService]);
}
```

```ts
class DocsSearchService extends Service {
  search$(query: string): LiveData<SearchResult[]>;
}
```

---

### `journal` module

Daily journal docs — one doc per date, auto-created.

```ts
export function configureJournalModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(JournalService, [JournalStore, DocsService, TemplateDocService]).store(JournalStore, [DocsService]).scope(DocScope).service(JournalDocService, [DocService, JournalService]);
}
```

```ts
class JournalService extends Service {
  // Get or create journal doc for a date:
  getJournalDocId(date: MaybeDate): string | undefined;
  getJournalByDate(date: MaybeDate): DocRecord | undefined;
  getOrCreateJournal(date: MaybeDate): DocRecord;

  journalDates$: LiveData<string[]>; // all dates that have a journal
  isJournal$(docId: string): LiveData<boolean>;
}

class JournalDocService extends Service {
  // Within a DocScope — is the current doc a journal?
  isJournal$: LiveData<boolean>;
  journalDate$: LiveData<string | undefined>;
}

const JOURNAL_DATE_FORMAT = 'YYYY-MM-DD';
```

---

### `share-doc` module

Manage public sharing of docs.

```ts
export function configureShareDocsModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(ShareDocsListService, [WorkspaceService]).store(ShareDocsStore, [WorkspaceServerService]).entity(ShareDocsList, [WorkspaceService, ShareDocsStore, WorkspaceLocalCache]).scope(DocScope).entity(ShareInfo, [ShareStore, DocService, GlobalCache]).store(ShareStore, [WorkspaceServerService]).service(ShareInfoService);
}
```

```ts
class ShareInfoService extends Service {
  shareInfo: ShareInfo; // entity
}

class ShareInfo extends Entity {
  isShared$: LiveData<boolean>;
  shareMode$: LiveData<'page' | 'edgeless' | undefined>;
  publishedDoc$: LiveData<{ published: boolean; publishMode: string } | null>;

  enableShare(mode: 'page' | 'edgeless'): Promise<void>;
  disableShare(): Promise<void>;
}
```

---

### `peek-view` module

Floating preview modal — opens a doc in a peek panel without leaving the current view.

```ts
export function configurePeekViewModule(framework: Framework) {
  framework.scope(WorkspaceScope).service(PeekViewService).entity(PeekViewEntity, [WorkbenchService]);
}
```

```ts
class PeekViewService extends Service {
  peekView: PeekViewEntity;
}

class PeekViewEntity extends Entity {
  isOpen$: LiveData<boolean>;
  target$: LiveData<PeekViewTarget | null>;

  open(target: PeekViewTarget): void;
  close(): void;
}
```

---

## 5. Core modules in depth

### `workspace-engine` module (`workspace/entities/engine.ts`)

The engine connects the workspace to nbstore. It wraps the nbstore `StoreClient` and provides typed frontends for doc, blob, indexer, and awareness operations.

```ts
// engine.start() opens the nbstore store:
const { store, dispose } = this.nbstoreService.openStore(`workspace:${flavour}:${id}`, engineWorkerInitOptions);
this.client = store;
this.eventBus.emit(WorkspaceEngineBeforeStart, this);

// Priority loading: root doc is loaded first at priority 100
this.doc.addPriority(rootDoc.guid, 100);

// Battery save mode can be enabled for non-local cloud workspaces:
if (flags.enable_battery_save_mode && flavour !== 'local') {
  store.enableBatterySaveMode();
}
```

**Events:**

- `WorkspaceEngineBeforeStart` — emitted just before the engine starts syncing
- `WorkspaceInitialized` — emitted when workspace data is ready

---

### `doc` module — `Doc` entity construction

When a `DocScope` is created (via `docsService.createDoc()` or opening a doc by id), the `Doc` entity is instantiated with:

1. A reference to the `BlockSuiteDoc` (from `docCollection.getDoc(id)`)
2. Its `DocRecord` (lightweight metadata)
3. Auto-registration with engine at priority 100 for both doc sync and indexing

The `afterTransaction` Yjs listener calls `setUpdatedAt` throttled at 1s to avoid writing on every keystroke.

---

### `db` module — Yjs ORM internals

`WorkspaceDB` tables are backed by Yjs `Y.Map` instances inside the root doc. Each table maps `id → record`. The ORM uses `@toeverything/infra`'s `ORM` class which provides:

- `findAll$()` — `LiveData` of all records (updates reactively)
- `find$(id)` — `LiveData` of one record or `null`
- `create(data)` — inserts a new record
- `update(id, patch)` — merges a partial update
- `delete(id)` — removes a record

This makes tags, folders, favorites, and doc properties work offline with CRDT conflict resolution.

---

## 6. Cloud module — full breakdown

The cloud module is the largest in @affine/core. It uses `ServerScope` for services that are per-server-instance and `WorkspaceScope` for workspace-level cloud concerns.

```ts
export function configureCloudModule(framework: Framework) {
  configureDefaultAuthProvider(framework);   // sets up AuthProvider impl

  // Global / ServerScope services:
  framework
    .service(ServersService, [ServerListStore, ServerConfigStore])
    .service(DefaultServerService, [ServersService])
    .store(ServerListStore, [GlobalStateService])
    .store(ServerConfigStore)
    .entity(Server, [ServerListStore])
    .scope(ServerScope)
      .service(ServerService, [ServerScope])
      .service(FetchService, [ServerService])
      .service(EventSourceService, [ServerService])
      .service(GraphQLService, [FetchService])
      .service(CaptchaService, [ServerService, FetchService, ValidatorProvider?])
      .service(AuthService, [FetchService, AuthStore, UrlService, GlobalDialogService])
      .store(AuthStore, [FetchService, GraphQLService, GlobalState, ServerService, AuthProvider])
      .entity(AuthSession, [AuthStore])
      .service(SubscriptionService, [SubscriptionStore])
      .entity(Subscription, [AuthService, ServerService, SubscriptionStore])
      .entity(SubscriptionPrices, [ServerService, SubscriptionStore])
      .service(UserQuotaService)
      .entity(UserQuota, [AuthService, UserQuotaStore])
      .service(UserFeatureService)
      .entity(UserFeature, [AuthService, UserFeatureStore])
      .service(UserCopilotQuotaService)
      .entity(UserCopilotQuota, [AuthService, UserCopilotQuotaStore, ServerService])
      .service(InvoicesService)
      .entity(Invoices, [InvoicesStore])
      .service(PublicUserService, [PublicUserStore])
      .service(UserSettingsService, [UserSettingsStore])
      .service(AccessTokenService, [AccessTokenStore])
      .service(InvitationService, [AcceptInviteStore, InviteInfoStore])
      .service(SelfhostGenerateLicenseService, [SelfhostGenerateLicenseStore]);

  // WorkspaceScope cloud services:
  framework
    .scope(WorkspaceScope)
      .service(WorkspaceServerService)
      .service(DocCreatedByService, [WorkspaceServerService])
      .service(WorkspaceSubscriptionService, [WorkspaceServerService])
      .entity(WorkspaceSubscription, [WorkspaceService, WorkspaceServerService])
      .service(WorkspaceInvoicesService)
      .entity(WorkspaceInvoices, [WorkspaceService, WorkspaceServerService])
      .service(SelfhostLicenseService, [SelfhostLicenseStore, WorkspaceService])
      .service(BlocksuiteWriterInfoService, [WorkspaceServerService])
      .service(DocCreatedByUpdatedBySyncService, [
        WorkspaceService, DocsService, WorkspacePermissionService,
        DocCreatedByUpdatedBySyncStore,
      ])
    .scope(DocScope)
      .service(DocUpdatedByService, [WorkspaceServerService])
      .service(CloudDocMetaService)
      .entity(CloudDocMeta, [CloudDocMetaStore, DocService, GlobalCache]);
}
```

### Key cloud services

**`ServersService`** — global registry of all server instances:

```ts
class ServersService extends Service {
  servers$: LiveData<Server[]>;
  getServer(baseUrl: string): Server;
  addServer(baseUrl: string): Promise<Server>;
  removeServer(serverId: string): void;
}
```

**`DefaultServerService`** — provides `affine.pro` (or `localhost` for development):

```ts
class DefaultServerService extends Service {
  server: Server; // the default AFFiNE cloud server entity
}
```

**`ServerService`** — per-ServerScope, provides config for the current server:

```ts
class ServerService extends Service {
  server: Server; // entity with serverConfig$, baseUrl, etc.
}
```

**`FetchService`** — HTTP client for a specific server (injects auth headers):

```ts
class FetchService extends Service {
  fetch(path: string, init?: RequestInit): Promise<Response>;
  // Automatically prefixes path with server baseUrl
  // Injects access token from AuthStore if available
}
```

**`GraphQLService`** — typed GraphQL client:

```ts
class GraphQLService extends Service {
  gql<Q extends GraphQLQuery>(options: { query: Q; variables?: Q['variables'] }): Promise<Q['response']>;
}
```

**`AuthService`** — authentication state machine:

```ts
@OnEvent(ApplicationFocused, e => e.onApplicationFocused)
@OnEvent(ServerStarted, e => e.onServerStarted)
class AuthService extends Service {
  session: AuthSession; // entity

  // Sign-in methods:
  sendEmailMagicLink(email, verifyToken?, challenge?, redirectUrl?): Promise<void>;
  signInMagicLink(email, token, byLink?): Promise<void>;
  oauthPreflight(provider, client, redirectUrl?): Promise<Record<string, string>>;
  signInOauth(code, state, provider): Promise<{ redirectUri?: string }>;
  signInPassword(credential: { email; password; verifyToken?; challenge? }): Promise<void>;
  signInOpenAppSignInCode(code: string): Promise<void>;
  createOpenAppSignInCode(): Promise<string>; // for cross-device sign-in

  // Sign-out / account:
  signOut(): Promise<void>;
  deleteAccount(): Promise<void>;

  // Auto-revalidation on app focus and server start
}
```

**`AuthSession`** entity — reactive auth state:

```ts
class AuthSession extends Entity {
  session$: LiveData<AuthSessionUnauthenticated | AuthSessionAuthenticated>;
  status$: LiveData<'authenticated' | 'unauthenticated'>;
  account$: LiveData<AuthAccountInfo | null>;
  isRevalidating$: LiveData<boolean>;

  revalidate(): void; // triggers effect to refresh from server
  waitForAuthenticated(signal?: AbortSignal): Promise<AuthSessionAuthenticated>;
}

interface AuthAccountInfo {
  id: string;
  label: string;
  email?: string;
  info?: AccountProfile | null;
  avatar?: string | null;
}
```

Auth events (listened to via `@OnEvent`):

- `AccountLoggedIn` — emitted when `account$` goes from null to a value
- `AccountLoggedOut` — emitted when `account$` goes to null
- `AccountChanged` — emitted on any account identity change

**`SubscriptionService` + `Subscription` entity:**

```ts
class Subscription extends Entity {
  pro$: LiveData<SubscriptionPlan | null>;
  team$: LiveData<SubscriptionPlan | null>;
  ai$: LiveData<SubscriptionPlan | null>;
  isLoading$: LiveData<boolean>;

  refresh(): void;
}
```

**`UserQuotaService` + `UserQuota` entity:**

```ts
class UserQuota extends Entity {
  quota$: LiveData<{ limit: number; used: number; humanReadable: QuotaHumanReadable } | null>;
  isLoading$: LiveData<boolean>;
}
```

**`WorkspaceServerService`** — within WorkspaceScope, links a workspace to its cloud server:

```ts
class WorkspaceServerService extends Service {
  server: Server | null; // null for local workspaces
  serverService: ServerService | null;
}
```

**`DocCreatedByUpdatedBySyncService`** — syncs `createdBy`/`updatedBy` fields on docs by listening to workspace doc events and writing to doc properties.

**`CloudDocMeta`** entity — per-doc cloud metadata (edit status, public links, etc.):

```ts
class CloudDocMeta extends Entity {
  editedBy$: LiveData<{ users: PublicUserInfo[] } | null>;
  isPublic$: LiveData<boolean>;
}
```

---

## 7. Workbench module — browser vs desktop

The workbench is the main window layout: sidebar, views (tabs), and their navigation state.

```ts
export function configureWorkbenchCommonModule(services: Framework) {
  services.scope(WorkspaceScope).service(WorkbenchService).entity(Workbench, [WorkbenchDefaultState, WorkbenchNewTabHandler, GlobalState]).entity(View).scope(ViewScope).service(ViewService, [ViewScope]).entity(SidebarTab);
}

// Browser (SPA — tabs are simulated, history in memory):
export function configureBrowserWorkbenchModule(services: Framework) {
  configureWorkbenchCommonModule(services);
  services
    .scope(WorkspaceScope)
    .impl(WorkbenchDefaultState, InMemoryWorkbenchDefaultState)
    .impl(WorkbenchNewTabHandler, () => BrowserWorkbenchNewTabHandler);
}

// Electron (real OS windows, state persisted to disk):
export function configureDesktopWorkbenchModule(services: Framework) {
  configureWorkbenchCommonModule(services);
  services.scope(WorkspaceScope).impl(WorkbenchDefaultState, DesktopWorkbenchDefaultState, [GlobalStateService, DesktopApiService]).impl(WorkbenchNewTabHandler, DesktopWorkbenchNewTabHandler, [DesktopApiService]).service(DesktopStateSynchronizer, [WorkbenchService, DesktopApiService, PeekViewService]);
}
```

**`Workbench`** entity — full API:

```ts
class Workbench extends Entity {
  // State:
  readonly activeViewIndex$: LiveData<number>;
  readonly basename$: LiveData<string>;
  readonly views$: LiveData<View[]>;

  // Computed:
  activeView$: LiveData<View>; // views$[activeViewIndex$]
  location$: LiveData<Location>; // activeView$.location$

  // Sidebar (persisted to GlobalState):
  sidebarOpen$: LiveData<boolean>;
  setSidebarOpen(open: boolean): void;
  openSidebar(): void;
  closeSidebar(): void;
  toggleSidebar(): void;

  sidebarWidth$: LiveData<number>; // default 320px
  setSidebarWidth(width: number): void;

  // Workspace selector dropdown:
  workspaceSelectorOpen$: LiveData<boolean>;
  setWorkspaceSelectorOpen(open: boolean): void;
  openWorkspaceSelector(): void;
  closeWorkspaceSelector(): void;
  toggleWorkspaceSelector(): void;

  // Navigation:
  active(indexOrView: number | View): void;
  updateBasename(basename: string): void;

  // View management:
  createView(at?: WorkbenchPosition, defaultLocation: To, active?: boolean): number;
  open(to: To, option?: WorkbenchOpenOptions): void;
  openDoc(id: string | DocOpenOptions, options?: WorkbenchOpenOptions): void;
  newTab(to: To, options?: { show?: boolean }): void;
  openCollections(options?: WorkbenchOpenOptions): void;
  openCollection(id: string, options?: WorkbenchOpenOptions): void;
  openTags(options?: WorkbenchOpenOptions): void;
  openTag(id: string, options?: WorkbenchOpenOptions): void;
  openAll(options?: WorkbenchOpenOptions): void;
  openTrash(options?: WorkbenchOpenOptions): void;
  openJournals(options?: WorkbenchOpenOptions): void;
}

type WorkbenchPosition = 'beside' | 'active' | 'head' | 'tail' | number;
type WorkbenchOpenOptions = {
  at?: WorkbenchPosition | 'new-tab';
  replaceHistory?: boolean;
  show?: boolean; // whether to activate the new tab
};
```

**`View`** entity — one browser-history-backed view pane:

```ts
class View extends Entity {
  readonly id: string;

  location$: LiveData<Location>;
  title$: LiveData<string>;
  icon$: LiveData<string | undefined>;

  // memento: scroll + editor state
  size$: LiveData<number>; // for split view width

  history: MemoryHistory;
  push(to: To): void;
  replace(to: To): void;
}
```

**Browser vs Desktop differences:**

| Concern   | Browser (`InMemoryWorkbenchDefaultState`) | Desktop (`DesktopWorkbenchDefaultState`)                |
| --------- | ----------------------------------------- | ------------------------------------------------------- |
| Tab state | Lives in memory                           | Persisted to `GlobalStateService` + Electron main       |
| New tab   | Opens in same window                      | Creates new Electron `BrowserWindow` tab                |
| Tab sync  | Not needed                                | `DesktopStateSynchronizer` pushes state to main process |
| History   | `MemoryHistory` per view                  | Same, but initialized from saved state                  |

---

## 8. BlockSuite integration

BlockSuite (the collaborative editor) is a separate Lit Web Components framework — **not React**. The integration lives in `src/blocksuite/`.

```
src/blocksuite/
  block-suite-editor/       ← AffineEditorContainer (Lit component)
  view-extensions/          ← AFFiNE-specific BlockSuite view extensions
  store-extensions/         ← Extends BlockSuite store with AFFiNE data
  manager/                  ← EditorManager: lifecycle + mounting
  editors/                  ← React wrappers for editor instances
  utils/                    ← Editor utility functions
  affine/                   ← AFFiNE-specific block configurations
  ai/                       ← AI copilot integration into editor
```

**Connecting BlockSuite to DI:**

```ts
// In Workspace entity — the docCollection wraps rootYDoc with BlockSuite:
get docCollection(): WorkspaceInterface {
  return new WorkspaceImpl({
    id: this.id,
    rootDoc: this.rootYDoc,
    blobSource: {
      get: key => this.engine.blob.get(key),
      set: (id, blob) => this.engine.blob.set({ key: id, data, mime }),
    },
    onLoadDoc: doc => this.engine.doc.connectDoc(doc),
    onLoadAwareness: awareness => this.engine.awareness.connectAwareness(awareness),
    onCreateDoc: docId => this.docs.createDoc({ id: docId, skipInit: true }).id,
  });
}
```

**React ↔ Lit bridge:**

```ts
// Used in src/blocksuite/editors/ to embed Lit editor in React:
import { createReactComponentFromLit } from '@affine/component/lit-react';
const EditorContainer = createReactComponentFromLit(AffineEditorContainer);
```

**View extensions** — how AFFiNE features plug into the editor:

```ts
// modules/xxx/view-extensions/
// Each extension registers itself with BlockSuite's extension system.
// Example: comment panel, AI button, doc info panel
```

---

## 9. Shared components

```
src/components/
  affine/             ← AFFiNE-specific app-level components
    affine-avatar/
    affine-error-boundary/
    affine-other-page/
    create-workspace/
    workspace-list/
  cloud/              ← Cloud-specific UI (sign-in, quota, subscription)
  comment/            ← Inline comment thread UI
  context/            ← React context providers
  explorer/           ← File tree / workspace explorer
  filter/             ← Collection filter builder
  guard/              ← Permission gates (render children only if allowed)
  hooks/              ← Shared React hooks
  member-selector/    ← Invite / member picker component
  mobile/             ← Mobile-specific components
  notification/       ← In-app notification toasts
  over-capacity/      ← Storage quota exceeded overlay
  page-detail-editor/ ← Main doc editing page wrapper
  page-list/          ← Doc list with sorting/filtering
  properties/         ← Doc properties panel
  providers/          ← React Provider trees (DI, theme, router)
  pure/               ← Stateless presentational components
```

Key component files:

- `src/components/page-detail-editor.tsx` — mounts the BlockSuite editor, handles mode switching (page / edgeless)
- `src/components/page-list/` — virtual-scrolled list of docs with tag/sort/filter
- `src/components/providers/` — `FrameworkRoot`, `ThemeProvider`, `I18nProvider`

---

## 10. Patterns reference

### LiveData — reactive state

```ts
// In a service or entity:
class DocsService extends Service {
  // Primary reactive value:
  readonly docs$ = new LiveData<Doc[]>([]);

  // Derived (never stale, always in sync):
  readonly trashDocs$ = LiveData.computed(get => get(this.docs$).filter(d => d.trash$.value));

  // From an Observable:
  readonly loading$ = LiveData.from(someObservable$, false);

  // Transform:
  readonly count$ = this.docs$.map(docs => docs.length);
}

// In React:
const docs = useLiveData(docsService.docs$);
const count = useLiveData(docsService.count$);
```

### `effect()` — side effects outside React

```ts
// In a Service constructor (auto-cleaned by disposables):
this.disposables.push(
  effect(
    switchMap(() => this.authService.session.account$),
    switchMap(account => (account ? fromPromise(() => loadWorkspaces(account.id)) : of([]))),
    tap(workspaces => this.workspaces$.next(workspaces))
  )
);
```

### `@OnEvent` decorator — event-driven service logic

```ts
// Declare listener at class level:
@OnEvent(ApplicationFocused, e => e.onApplicationFocused)
@OnEvent(ServerStarted, e => e.onServerStarted)
class AuthService extends Service {
  private onApplicationFocused() {
    this.session.revalidate();
  }
  private onServerStarted() {
    this.session.revalidate();
  }
}

// Emit events from any service:
this.eventBus.emit(AccountLoggedIn, account);
```

### Service disposables

All subscriptions must be pushed to `this.disposables` — cleaned up automatically when the scope containing the service is destroyed:

```ts
class MyService extends Service {
  constructor(private other: OtherService) {
    super();

    // RxJS subscription:
    this.disposables.push(other.value$.subscribe(v => this.handle(v)));

    // Unsubscribe function:
    this.disposables.push(() => {
      window.removeEventListener('resize', this.onResize);
    });
  }
}
```

### `.impl()` for platform swapping

```ts
// Define abstract provider in core:
abstract class NbstoreProvider {
  abstract openStore(universalId: string): StoreClient;
}

// Web provides IndexedDB-backed impl:
framework.impl(NbstoreProvider, BrowserNbstoreProvider);

// Electron provides SQLite-backed impl:
framework.impl(NbstoreProvider, ElectronNbstoreProvider, [DesktopApiService]);

// Mobile Capacitor impl:
framework.impl(NbstoreProvider, CapacitorNbstoreProvider);
```

### `ObjectPool` — shared resource management

Used internally for open docs, open collections, etc. to avoid duplicate instances:

```ts
// Open a doc (returns existing instance if already open):
const doc = docsService.pool.open(docId);
// Close when done (decrements ref count, destroys when count reaches 0):
doc.release();
```

### Optional multi-providers `[Provider]`

Services can accept zero or more implementations of a provider:

```ts
.service(DocsService, [DocsStore, DocPropertiesStore, [DocCreateMiddleware]])
//                                                     ^ array = multi-provider

// DocsService constructor receives all registered DocCreateMiddleware impls:
constructor(
  private readonly store: DocsStore,
  private readonly propsStore: DocPropertiesStore,
  private readonly middlewares: DocCreateMiddleware[]   // ← array
) {}
```

### Factory function DI

For services needing custom construction logic:

```ts
framework.scope(ServerScope).service(CaptchaService, f => {
  return new CaptchaService(
    f.get(ServerService),
    f.get(FetchService),
    f.getOptional(ValidatorProvider) // optional dep
  );
});
```

---

## 11. How apps wire core

Every app entry point does the same three steps:

```ts
// 1. Create the DI framework
const framework = new Framework();

// 2. Register all core modules
configureStorageModule(framework);
configureLifecycleModule(framework);
configureFeatureFlagModule(framework);
configureWorkspaceModule(framework);
configureDocModule(framework);
configureWorkspaceDBModule(framework);
configureEditorModule(framework);
configureCloudModule(framework);
configureTagModule(framework);
configureCollectionModule(framework);
configureFavoriteModule(framework);
configureOrganizeModule(framework);
configureWorkbenchCommonModule(framework);
configureDialogModule(framework);
configureNavigationModule(framework);
configureQuickSearchModule(framework);
configureDocsSearchModule(framework);
configureJournalModule(framework);
configureThemeModule(framework);
configurePermissionsModule(framework);
configurePeekViewModule(framework);
configureShareDocsModule(framework);
configureNotificationModule(framework);
// ... 40+ more

// 3. Inject platform-specific implementations
// Web app:
configureBrowserWorkbenchModule(framework);
configureLocalStorageStateStorageImpls(framework);
configureBrowserWorkspaceFlavours(framework);
framework.impl(NbstoreProvider, BrowserNbstoreProvider);
framework.impl(PopupWindowProvider, BrowserPopupWindowProvider);

// Electron replaces:
configureDesktopWorkbenchModule(framework);
configureElectronStateStorageImpls(framework);
framework.impl(NbstoreProvider, ElectronNbstoreProvider);
framework.impl(DesktopApiService, ElectronDesktopApiService);
framework.impl(PopupWindowProvider, ElectronPopupWindowProvider);

// Mobile Capacitor adds:
framework.impl(HapticProvider, CapacitorHapticProvider);
framework.impl(VirtualKeyboardProvider, CapacitorVirtualKeyboardProvider);
framework.impl(NbstoreProvider, CapacitorNbstoreProvider);

// 4. Build the provider tree and wrap React:
const provider = framework.provider();

<FrameworkRoot framework={provider}>
  <RouterProvider router={router} />
</FrameworkRoot>
```

The DI container resolves all dependencies lazily — services are instantiated only when first accessed.
