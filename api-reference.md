---
title: API reference
---

# API reference

[← Back to index](./)

Everything exported from `@vrcnext/plugin-api` (version **0.2.0**). Types are authoritative in
the package; this is the map. The package is types plus a little pure logic — manifest parsing,
the permission vocabulary, semver, settings defaults — with no DOM and no host dependency, so
you can unit-test against it.

## Entry

```ts
function definePlugin<const S extends SettingsSchema>(plugin: VrcnextPlugin<S>): VrcnextPlugin<S>;

interface VrcnextPlugin<S> {
  readonly id: PluginId;               // must equal "id" in plugin.json
  readonly settings?: S;
  activate(ctx: PluginContext<S>): void | Promise<void>;
  deactivate?(): void | Promise<void>;
}
```

## `PluginContext`

| Member | Type | Permission |
| :--- | :--- | :--- |
| `id` | `PluginId` | — |
| `version` | `string` | — |
| `logger` | `Logger` | — |
| `settings` | `SettingsStore<S>` | — |
| `permissions` | `PermissionsApi` | — |
| `events` | `EventBus` | `host:events`, event names checked against `events` |
| `bridge` | `Bridge` | `host:actions` (`send`, `request`; action names checked against `actions`), `host:intercept` (`interceptOutbound`) |
| `http` | `HttpApi` | `network`, hosts checked against `hosts` |
| `ui` | `UiApi` | — |
| `notifications` | `NotificationsApi` | `notifications` |
| `osc` | `OscApi` | `osc` |
| `native` | `NativeApi` | `native` |
| `gameLog` | `GameLogApi` | `gamelog` |
| `deepLinks` | `DeepLinkApi` | `host:events` with `openDeepLink` in `events` |
| `router` | `RouterApi` | `routes` |
| `contextMenu` | `ContextMenuApi` | `context-menu` |
| `clipboard` | `ClipboardApi` | `clipboard` |
| `disposables` | `DisposableBag` | — |
| `signal` | `AbortSignal` | — |

A gated member called without its category throws `PermissionError`; a concrete target the user
has not confirmed prompts first. See [Permissions](permissions.md).

## Permissions

```ts
type Permission = 'host:events' | 'host:actions' | 'host:intercept' | 'network' | 'notifications'
  | 'native' | 'osc' | 'gamelog' | 'context-menu' | 'routes' | 'clipboard';
type PermissionTone = 'low' | 'medium' | 'high';
interface PermissionInfo { readonly description: string; readonly tone: PermissionTone }

const PERMISSIONS: Readonly<Record<Permission, PermissionInfo>>;
const PERMISSION_NAMES: readonly Permission[];
function isPermission(value: unknown): value is Permission;
function permissionInfo(permission: Permission): PermissionInfo;

class PermissionError extends Error {
  readonly permission: Permission;
  readonly target: string | undefined;   // host, action, event or service/method; undefined for a category
}

interface PermissionsApi {
  has(permission: Permission): boolean;              // cheap; safe per event
  request(permission: Permission): Promise<boolean>; // optionalPermissions only; rejects otherwise
}
```

## Manifest and identifiers

```ts
const MANIFEST_FILENAME = 'plugin.json';
const MANIFEST_LIMITS = { descriptionChars: 200, tags: 8 };
const PLUGIN_TAGS: readonly string[];                 // suggested tags; any string is accepted
const PLUGIN_ID_PATTERN = /^[a-z0-9][a-z0-9-]{1,39}$/;

type PluginId;                                        // branded string
function isPluginId(value: string): value is PluginId;
function parsePluginId(value: string): PluginId | undefined;

interface PluginSummary {
  readonly id: PluginId; readonly name: string; readonly version: string;
  readonly description: string; readonly tags: readonly PluginTag[];
  readonly permissions: readonly Permission[];
  readonly optionalPermissions: readonly Permission[];
  readonly actions: readonly string[]; readonly events: readonly string[];
  readonly hosts: readonly string[];
}
interface PluginManifest extends PluginSummary {
  readonly apiVersion: string; readonly author?: string; readonly homepage?: string;
  readonly dependencies?: readonly PluginId[];        // not accepted by the bridge yet
}
function parsePluginManifest(raw: unknown): ManifestParseResult;   // { manifest | undefined, errors }
```

## Semver

The slice this project needs: plain `MAJOR.MINOR.PATCH` versions, and ranges that are exact,
`^`, `~` or a space-separated comparator list.

```ts
type Version = readonly [major: number, minor: number, patch: number];
function parseVersion(text: string): Version | undefined;
function isVersion(text: string): boolean;
function compareVersions(a: Version, b: Version): -1 | 0 | 1;
function parseRange(text: string): ((version: Version) => boolean) | undefined;
function isRange(text: string): boolean;
function satisfies(version: string, range: string): boolean;   // malformed input never satisfies
```

## Settings

`SettingsSchema` · `SettingsValues<S>` · `SettingsStore<S>` · `SettingSpec`
`BooleanSetting` · `NumberSetting` · `StringSetting` · `ColorSetting` · `SelectSetting<V>` ·
`SelectOption<V>` · `defaultsFor()` · `coerceSetting()`

```ts
interface SettingsStore<S> {
  readonly values: SettingsValues<S>;
  get<K extends keyof S>(key: K): SettingsValues<S>[K];
  set<K extends keyof S>(key: K, value: SettingsValues<S>[K]): Promise<void>;
  reset(): Promise<void>;
  onChange(listener: (values: SettingsValues<S>) => void): () => void;
}
```

## Events

```ts
interface EventBus {
  on<T extends string>(type: T, listener: EventListener<T>): () => void;
  once<T extends string>(type: T, listener: EventListener<T>): () => void;
  onAny(listener: (envelope: HostEnvelope) => void): () => void;   // prompts: "listen to every VRCNext event"
  next<T extends string>(type: T, signal?: AbortSignal): Promise<EventPayload<T>>;
}
```

`VrcnextEventMap` — verified payloads: `setPlatform` · `log` · `toast` · `dbMigrationProgress` ·
`openDeepLink` · `friendTimelineEvent` · `customThemes`. Anything else is `unknown`.
`KnownEventType` · `EventPayload<T>` · `EventListener<T>` · `HostEnvelope` · `LogColor` · `LOG_COLORS`

## Bridge (VRCNext actions)

```ts
interface Bridge {
  send(action: ActionName, args?: ActionArgs): void;
  request<T extends string>(action: ActionName, args: ActionArgs | undefined,
    options: RequestOptions & { readonly expect: T }): Promise<EventPayload<T>>;
  interceptOutbound(fn: (action: ActionName, raw: string) => boolean | undefined): () => void;
}
interface RequestOptions { readonly expect: string; readonly timeoutMs?: number; readonly signal?: AbortSignal }
```

## HTTP and clipboard

```ts
interface HttpApi {
  /** The only fetch a plugin has. Always carries the plugin's abort signal. */
  fetch(url: string | URL, init?: RequestInit): Promise<Response>;
}
interface ClipboardApi {
  writeText(text: string): Promise<void>;
  readText(): Promise<string>;
}
```

## UI

```ts
interface UiApi {
  addNavTab(options: NavTabOptions): PanelHandle;
  addDashboardCard(options: DashboardCardOptions): PanelHandle;
  addSettingsCard(options: SettingsCardOptions): PanelHandle;
  injectCss(css: string): PanelHandle;
  toast(options: ToastOptions): void;
  readonly kit: UiKit;
  createPanelLayout(): HTMLElement;
  createCard(title: string, icon: IconName): HTMLElement;
  createToggleRow(label: string, checked: boolean, onChange: (next: boolean) => void): HTMLElement;
}
```

`NavTabOptions` · `DashboardCardOptions` · `SettingsCardOptions` · `PanelHandle` · `IconName` ·
`ToastOptions`

## UI kit

`ctx.ui.kit` — declarative builders that emit VRCNext's own markup. See [UI injection](ui.md).

```ts
type UiChild = Node | string | false | null | undefined;   // falsy children are dropped
type UiBadgeTone = 'ok' | 'warn' | 'err' | 'accent' | 'cyan' | 'neutral';

interface UiKit {
  layout(...children: readonly UiChild[]): HTMLElement;
  grid(children: readonly UiChild[], options?: { min?: number }): HTMLElement;
  pair(first: UiChild, second: UiChild): HTMLElement;
  card(options: { title?: string; icon?: IconName; children?: readonly UiChild[]; span?: number }): HTMLElement;
  statusCard(options: { tone: 'online' | 'warn' | 'offline'; label: string; action?: Node }): HTMLElement;
  section(label: string, children: readonly UiChild[]): DocumentFragment;

  row(options: { label: string; value?: UiChild; detail?: string }): HTMLElement;
  toggleRow(options: { label: string; value: boolean; onChange: (next: boolean) => void; detail?: string }): HTMLElement;
  buttonRow(...children: readonly UiChild[]): HTMLElement;
  button(options: { label: string; icon?: IconName; onClick: () => void; active?: boolean; disabled?: boolean; round?: boolean }): HTMLButtonElement;
  textField(options: { value: string; placeholder?: string; onCommit: (next: string) => void }): HTMLInputElement;
  dropdown(options: { options: readonly { value: string; label: string }[]; selected: string; onChange: (next: string) => void }): HTMLSelectElement;

  badge(tone: UiBadgeTone, text: string): HTMLElement;
  stat(options: { label: string; value: string; tone?: UiBadgeTone }): HTMLElement;
  description(text: string): HTMLElement;
  sectionLabel(text: string): HTMLElement;
  valueText(text: string): HTMLElement;
  emptyState(text: string): HTMLElement;

  setChildren(parent: Node, children: readonly UiChild[]): void;
}
```

## Notifications

```ts
interface NotificationsApi {
  toast(options: ToastOptions): void;
  notifToast(options: NotifToastOptions): void;
  desktop(options: DesktopNotifyOptions): void;   // tray + SteamVR overlay; Windows only
  readonly desktopAvailable: boolean;
  confirm(options: ConfirmOptions): Promise<boolean>;
}
```

`NOTIFY_ACCENTS` = `'accent' | 'info' | 'ok' | 'warn' | 'err'`
`NOTIF_TOAST_KINDS` = `'invite' | 'friendRequest' | 'notification'`

## Context menu

```ts
interface ContextMenuApi {
  contribute(provider: ContextMenuProvider): () => void;
  contributeFor(selector: string, provider: ContextMenuProvider): () => void;
  open(x: number, y: number, entries: readonly ContextMenuEntry[]): void;
}
```

`ContextMenuEntry` = `ContextMenuItem | ContextMenuSubmenu | ContextMenuDivider` · `ContextMenuTarget`

## OSC

```ts
interface OscApi {
  readonly available: boolean;   // false on Linux — VRCNext filters every osc* action
  connect(): void;
  disconnect(): void;
  send(name: string, kind: 'bool', value: boolean): void;
  send(name: string, kind: 'int' | 'float', value: number): void;
  sendRaw(address: string, kind: 'bool', value: boolean): void;
  sendRaw(address: string, kind: 'int' | 'float', value: number): void;
  onParam(listener: (event: OscParamEvent) => void): () => void;
  onAvatarChange(listener: (event: OscAvatarChangeEvent) => void): () => void;
}
```

`OSC_VALUE_KINDS` · `OscValueKind` · `OscValue` · `OscParamEvent` · `OscAvatarChangeEvent`

## VRCNext Bridge

```ts
interface NativeApi {
  targets(): Promise<readonly NativeTarget[]>;
  notify(options: NativeNotifyOptions): Promise<NativeNotifyResult>;
  call(service: string, method: string, params?: unknown): Promise<unknown>;
}

interface NativeTarget {
  readonly name: string;                            // 'wayvr', 'freedesktop', …
  readonly description: string;
  readonly health: 'up' | 'unknown' | 'down';
  readonly honours: readonly string[];              // which fields this target actually uses
}

interface NativeNotifyOptions extends NativeNotifyFields {
  readonly title: string;                           // required
  readonly sinks?: readonly string[];               // omit for every target
  readonly overrides?: Readonly<Record<string, NativeNotifyOverride>>;
}

// Shared by a request and its per-target overrides.
interface NativeNotifyFields {
  readonly content?: string;
  readonly timeoutSecs?: number;
  readonly icon?: string;
  readonly useBase64Icon?: boolean;
  readonly sourceApp?: string;
  readonly sound?: boolean;
  readonly volume?: number;
  readonly audioPath?: string;
  readonly height?: number;      // VR overlays only
  readonly opacity?: number;     // VR overlays only
  readonly urgency?: NativeUrgency;  // 'low' | 'normal' | 'critical'; desktop daemons only
  readonly alwaysShow?: boolean; // VR overlays only
}

interface NativeNotifyResult {
  readonly ok: boolean;          // true if at least one target accepted
  readonly delivered: readonly string[];
  readonly failed: readonly { readonly sink: string; readonly error: string }[];
}
```

The bridge is connected whenever a plugin runs, so there is no `available`, `status` or `probe`.
`notify()` resolves with `ok: false` rather than rejecting when no target accepted; `call()`
rejects with an error carrying the bridge's own `code`. See [VRCNext Bridge](native-companion.md).

## Game log

```ts
interface GameLogApi {
  on(listener: (entry: GameLogEntry) => void): () => void;
  onType(type: string, listener: (entry: GameLogEntry) => void): () => void;
  history(signal?: AbortSignal): Promise<readonly GameLogEntry[]>;
}
```

## Routes and deep links

```ts
interface RouterApi {
  readonly base: URL;
  route(method: string, pattern: string, handler: RouteHandler): () => void;
  get(pattern: string, handler: RouteHandler): () => void;
  post(pattern: string, handler: RouteHandler): () => void;
  fetch(input: string | URL, init?: RequestInit): Promise<Response>;
}

interface DeepLinkApi {
  on(listener: (event: DeepLinkEvent) => boolean | undefined): () => void;
  onPrefix(prefix: DeepLinkPrefix, listener: (event: DeepLinkEvent) => boolean | undefined): () => void;
}
```

`DEEP_LINK_PREFIXES` = `'usr' | 'avtr' | 'wrld' | 'grp' | 'inst' | 'instjoin'` · `RouteRequest` · `RouteHandler`

## Disposal and logging

`DisposableBag` · `Disposable` · `DisposeFn`
`Logger` · `LogLevel` · `LOG_LEVELS`

[← Limitations](limitations.md) · [Back to index](./)
