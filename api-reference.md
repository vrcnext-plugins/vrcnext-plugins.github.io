---
title: API reference
---

# API reference

[← Back to index](./)

Everything exported from `@vrcnext/plugin-api`. Types are authoritative in the package; this is
the map.

## Entry

```ts
function definePlugin<const S extends SettingsSchema>(plugin: VrcnextPlugin<S>): VrcnextPlugin<S>;

interface VrcnextPlugin<S> {
  readonly id: PluginId;
  readonly settings?: S;
  activate(ctx: PluginContext<S>): void | Promise<void>;
  deactivate?(): void | Promise<void>;
}
```

## `PluginContext`

| Member | Type |
| :--- | :--- |
| `id` | `PluginId` |
| `version` | `string` |
| `logger` | `Logger` |
| `settings` | `SettingsStore<S>` |
| `events` | `EventBus` |
| `bridge` | `Bridge` |
| `ui` | `UiApi` |
| `notifications` | `NotificationsApi` |
| `osc` | `OscApi` |
| `native` | `NativeApi` |
| `gameLog` | `GameLogApi` |
| `deepLinks` | `DeepLinkApi` |
| `router` | `RouterApi` |
| `contextMenu` | `ContextMenuApi` |
| `disposables` | `DisposableBag` |
| `signal` | `AbortSignal` |

## Identifiers

`PluginId` · `PluginKey` · `RepoId` — branded strings.
`isPluginId()` · `parsePluginId()` · `makePluginKey()` · `makeRepoId()`

## Settings

`SettingsSchema` · `SettingsValues<S>` · `SettingsStore<S>` · `SettingSpec`
`BooleanSetting` · `NumberSetting` · `StringSetting` · `SelectSetting<V>` · `SelectOption<V>`
`defaultsFor()` · `coerceSetting()`

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
  onAny(listener: (envelope: HostEnvelope) => void): () => void;
  next<T extends string>(type: T, signal?: AbortSignal): Promise<EventPayload<T>>;
}
```

`VrcnextEventMap` — verified payloads: `setPlatform` · `log` · `toast` · `dbMigrationProgress` ·
`openDeepLink` · `friendTimelineEvent` · `customThemes`. Anything else is `unknown`.

## Bridge

```ts
interface Bridge {
  send(action: ActionName, args?: ActionArgs): void;
  request<T extends string>(action, args, options: RequestOptions & { expect: T }): Promise<EventPayload<T>>;
  interceptOutbound(fn: (action: ActionName, raw: string) => boolean | undefined): () => void;
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
  createCard(title: string, icon: IconName): HTMLElement;
  createToggleRow(label: string, checked: boolean, onChange: (next: boolean) => void): HTMLElement;
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

`ContextMenuEntry` = `ContextMenuItem | ContextMenuSubmenu | ContextMenuDivider`

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
  statusCard(options: { online: boolean; label: string; action?: Node }): HTMLElement;
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

## Native companion

```ts
interface NativeApi {
  readonly available: boolean;        // synchronous snapshot of the last probe
  readonly ready: Promise<boolean>;   // the boot probe — await this inside activate()
  readonly endpoint: string;
  probe(): Promise<boolean>;
  describe(): Promise<NativeDescription | undefined>;
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
  readonly urgency?: 'low' | 'normal' | 'critical';  // desktop daemons only
  readonly alwaysShow?: boolean; // VR overlays only
}

interface NativeNotifyResult {
  readonly ok: boolean;          // true if at least one target accepted
  readonly delivered: readonly string[];
  readonly failed: readonly { readonly sink: string; readonly error: string }[];
}
```

Every method degrades cleanly when the companion is absent — `notify()` resolves with
`ok: false` rather than rejecting. Only `call()` rejects, with an error carrying the companion's
own `code` and `status`. See [Native companion](native-companion.md).

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

## Manifest and disposal

`parseRepoManifest()` · `RepoManifest` · `PluginManifest` · `ManifestParseResult`
`MANIFEST_FILENAME` · `MANIFEST_FORMAT_VERSION`
`DisposableBag` · `Disposable` · `DisposeFn`
`Logger` · `LogLevel` · `LOG_LEVELS` · `LogColor` · `LOG_COLORS`

[← Limitations](limitations.md) · [Back to index](./)
