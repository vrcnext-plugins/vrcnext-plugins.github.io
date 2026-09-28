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
| `vrchat` | `VrchatApi` | `vrchat` |
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
  | 'native' | 'osc' | 'gamelog' | 'vrchat' | 'context-menu' | 'routes' | 'clipboard';
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
  readonly dependencies?: readonly PluginId[];        // activation order
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

## Time

```ts
type TimeInput = string | number | Date;
interface Timestamped { readonly timestamp: TimeInput }

function timeAgo(iso: string, now?: number): string;   // "just now", "3 minutes ago", "2 hours ago", "5 days ago"
function formatDuration(ms: number): string;           // "45 minutes", "3 hours", "5 days", "4 months", "2 years"
function newestFirst<T extends Timestamped>(items: readonly T[] | undefined): readonly T[];
```

`timeAgo(at, now?)` takes an ISO string, an epoch or a `Date`, and reports one unit: `just now`,
`3 minutes ago`, `2 hours ago`, `5 days ago`, `3 months ago`, `2 years ago`. `formatDuration(ms)`
picks the same coarse unit for a length of time rather than a point in one, and is `''` for zero.

`newestFirst(items)` orders anything carrying a `timestamp` — a `VrcTimelineEvent`, your own
records — newest first, and takes `undefined` for "nothing yet". An item whose timestamp cannot
be parsed is **left out** rather than sorted to 1970, since it cannot be placed in the order at
all. Use it instead of sorting a timeline yourself: `ctx.vrchat.userTimeline` arrives in no
guaranteed order.

## Templates

```ts
type TemplateValue = string | number | boolean | null | undefined | TemplateValue[] | { [key: string]: TemplateValue };
function renderTemplate(template: string, values: Record<string, TemplateValue>, options?: { dropEmptyLines?: boolean }): string;
function validateTemplate(template: string): TemplateError | undefined;
function templatePlaceholders(template: string): readonly string[];
class TemplateError extends Error {}
```

A small, safe template language for user-editable messages — Jinja-shaped, evaluated by a
walker over a fixed grammar, so a template can never run code or reach a prototype:

{% raw %}
```text
Player {name} joined                          {name} is short for {{ name }}
18+: {{ "yes" if ageVerified else "no" }}     Python-style conditional
Rejoin: {{ rejoin ? "yes" : "no" }}           C-style conditional
{% if inGroup == false %}NOT A MEMBER{% elif inGroup %}member{% else %}unknown{% endif %}
{{ pcRank | upper }} · {{ avatar | default: "unknown avatar" }} · {{ tags | join: "+" }}
```
{% endraw %}

Expressions: literals, dotted names, `== != < <= > >=`, `and or not` (or `&& || !`), `+`/`-`,
both conditional forms, and filters `upper lower capitalize trim length size default yesno join
replace truncate get` (`size` is `length` under Liquid's name; `get: "key"` reads a field). `elseif`
is accepted as a spelling of `elif`. A missing name renders empty; a line whose placeholders all rendered empty is
dropped (`dropEmptyLines`). A broken template throws `TemplateError`: an unknown filter, a tag
that is never closed ({% raw %}`{{name}`{% endraw %} included), or nesting deeper than `MAX_TEMPLATE_DEPTH` (32); `validateTemplate` reports
it without rendering. Small pure helpers plugins keep needing live in the api package, not in
plugins.

## Settings

`SettingsSchema` · `SettingsValues<S>` · `SettingsStore<S>` · `SettingSpec` · `SettingBase` ·
`InferSetting<S>` · `SettingPredicate` · `SelectOption<V>` · `ObjectToggle` · `TOGGLE_KEY` ·
`defaultsFor()` · `defaultOf()` · `coerceSetting()` · `settingFlag()` · `defineCustomSetting()`

See [Settings](settings.md) for the prose. Fifteen kinds:

```ts
type SettingSpec =
  | BooleanSetting | NumberSetting | StringSetting | ColorSetting | TimeSetting
  | SelectSetting<V> | MultiSelectSetting<V>
  | UserSetting | WorldSetting | AvatarSetting | GroupSetting | InstanceSetting   // EntitySetting
  | EmbedSetting | ObjectSetting<F> | ListSetting<I> | CustomSetting<T>;

interface SettingBase {
  readonly label: string;
  readonly description?: string;
  readonly hidden?: SettingPredicate;     // boolean, or (values) => boolean
  readonly disabled?: SettingPredicate;
}
```

| Spec | Extra fields | Value |
| :--- | :--- | :--- |
| `BooleanSetting` | — | `boolean` |
| `NumberSetting` | `min` `max` `step` `slider` `markers` `stickToMarkers` `integer` `unit` | `number` |
| `StringSetting` | `placeholder` `multiline` `maxLength` `format: 'text' \| 'password' \| 'url'` | `string` |
| `ColorSetting` | — | `'#rrggbb'` |
| `TimeSetting` | — | `'HH:MM'` |
| `SelectSetting<V>` | `options` | `V` |
| `MultiSelectSetting<V>` | `options` `min` `max` | `readonly V[]` |
| `EntitySetting` | `multiple` `scopes` `placeholder` | an id, or `readonly string[]` |
| `EmbedSetting` | `variables` | `EmbedTemplate` |
| `ObjectSetting<F>` | `fields` `collapsed` `toggle` (a switch on the header; state under `TOGGLE_KEY`, fields hidden while off) | `SettingsValues<F>` |
| `ListSetting<I>` | `item` `titleKey` `addLabel` `max` | `readonly SettingsValues<I>[]` |
| `CustomSetting<T>` | `coerce` `render` | `T` |

```ts
interface SettingsStore<S> {
  readonly values: SettingsValues<S>;
  get<K extends keyof S>(key: K): SettingsValues<S>[K];
  set<K extends keyof S>(key: K, value: SettingsValues<S>[K]): Promise<void>;
  reset(): Promise<void>;
  onChange(listener: (values: SettingsValues<S>) => void): () => void;
}

interface CustomSettingHost<T> {
  readonly value: T;
  setValue(next: T): Promise<void>;          // rejects when `coerce` refuses
  setError(message: string | undefined): void;
  onChange(listener: (value: T) => void): () => void;
}
```

`ENTITY_SCOPES` lists every scope per kind; `entityScopes(spec)` resolves a spec's to picker order,
and `isEntityId(kind, id)` checks one id.

### Discord embeds

`EmbedTemplate` · `EmbedField` · `EMBED_COLORS` · `EMBED_LIMITS` · `EMPTY_EMBED` ·
`completeEmbed()` · `coerceEmbed()`

```ts
function renderEmbed(template: EmbedTemplate, values: TemplateValues,
  options?: { at?: Date; onError?: (error: Error) => void }): DiscordEmbed | undefined;
function discordWebhookPayload(embed: DiscordEmbed,
  options?: { username?: string; content?: string }): DiscordWebhookPayload;
function parseEmbedColor(text: string): number | undefined;
```

Every text of the template is rendered with the template engine, empty parts are dropped, URLs
that are not `http(s)` are refused, everything is cut to Discord's limits, and `undefined` comes
back when nothing is left. `discordWebhookPayload` always sets `allowed_mentions: { parse: [] }`.

### Posting to a Discord webhook

```ts
function isDiscordWebhookUrl(url: string): boolean;
function webhookFailure(status: number): string;
function postWebhook(options: {
  http: HttpApi; logger: Logger; url: string;
  payload: DiscordWebhookPayload; label?: string;
}): Promise<{ ok: boolean; status?: number; error?: string }>;
```

`postWebhook` is the whole send: it refuses a URL that is not a `discord.com` webhook without
making a request, writes the full payload to the log at `debug` before it goes out, turns a
refusal into a sentence the user can act on, and never rejects — a failed channel should not take
its siblings down with it. `label` names the sender in each line, for a plugin with more than one
webhook.

Declare `discord.com` in your manifest's `hosts`. The webhook URL is a bearer credential for that
channel: it never appears in the log or in a failure message, so do not put it in one yourself.

The payload dump is not optional, and that is deliberate. When Discord does not render what you
meant, the only way to tell "Discord ignored it" from "I did not send it" is to see the bytes —
the two look identical from outside, a missing picture and a `204`. It is `debug`, so it is
written only while **Verbose debug logging** is on.

### Discord text

```ts
type DiscordTimeStyle = 't' | 'T' | 'd' | 'D' | 'f' | 'F' | 'R';

function discordCode(name: string): string;                                   // `Club Neon`
function discordTimestamp(at: TimeInput, style?: DiscordTimeStyle): string;    // <t:1790510400:R>
```

Discord applies its markdown to anything you interpolate, so a world called `**Club**` styles the
line it lands in. `discordCode` fences a name as a code span — and outgrows the backticks the name
itself contains, which is the only escape Discord honours.

`discordTimestamp` writes the `<t:…>` markup Discord renders in each reader's own timezone and
language. The default `R` style keeps saying "3 hours ago" correctly however long the message
sits in the channel, which a formatted clock time cannot; it is `''` for an instant it cannot
read, so the line drops instead of printing `NaN`.

## VRChat data

`ctx.vrchat`, permission `vrchat`. See [VRChat data](vrchat-data.md).

`VrchatApi` · `VrcUserSummary` · `VrcSelf` · `VrcUser` · `VrcFavoriteGroup` · `VrcModerationCounts` · `VrcAvatarSummary` · `VrcAvatar` ·
`VrcWorldSummary` · `VrcWorld` · `VrcGroupSummary` · `VrcGroup` · `VrcInstance` ·
`VrcInstanceUser` · `VrcFriendInstance` · `VrcTimelineEvent` · `VrcSearchPage<T>` ·
`VrcLookupOptions` · `VrcSearchOptions` · `PerformanceRank` · `PERFORMANCE_RANKS` · `rankIndex()`
· `rankLabel()` · `rankEmoji()` · `RANK_EMOJI` · `TrustRank` · `TRUST_RANKS` · `trustRank()` ·
`publicImageUrl()` · `isPublicImageUrl()`

```ts
interface VrchatApi {
  self(): VrcSelf | undefined;                                          // synchronous, and the fullest object here

  friends(o?: VrcLookupOptions): Promise<readonly VrcUserSummary[]>;
  favoriteFriends(o?): Promise<readonly VrcUserSummary[]>;
  favoriteFriendGroups(o?): Promise<readonly VrcFavoriteGroup[]>;       // { name, displayName, userIds, users }
  moderationCounts(o?): Promise<VrcModerationCounts>;                   // { blocked, muted, hiddenAvatar, interactOff, muteChat }
  recentPlayers(o?): Promise<readonly VrcUserSummary[]>;
  favoriteWorlds(o?): Promise<readonly VrcWorldSummary[]>;
  recentWorlds(o?): Promise<readonly VrcWorldSummary[]>;
  ownAvatars(o?): Promise<readonly VrcAvatarSummary[]>;
  favoriteAvatars(o?): Promise<readonly VrcAvatarSummary[]>;
  recentAvatars(o?): Promise<readonly VrcAvatarSummary[]>;
  myGroups(o?): Promise<readonly VrcGroupSummary[]>;
  currentInstance(o?): Promise<VrcInstance | undefined>;
  friendInstances(o?): Promise<readonly VrcFriendInstance[]>;

  user(id: string, o?): Promise<VrcUser | undefined>;
  userBasic(id: string, o?): Promise<VrcUserSummary | undefined>;
  userGroups(id: string, o?): Promise<readonly VrcGroupSummary[]>;
  userTimeline(id: string, o?): Promise<readonly VrcTimelineEvent[]>;
  instanceAvatar(userId: string, o?): Promise<{ avatarId: string; avatarName: string } | undefined>;
  avatar(id: string, o?): Promise<VrcAvatar | undefined>;
  world(id: string, o?): Promise<VrcWorld | undefined>;
  group(id: string, o?): Promise<VrcGroup | undefined>;

  searchUsers(query: string, o?: VrcSearchOptions): Promise<VrcSearchPage<VrcUserSummary>>;
  searchWorlds(query: string, o?: VrcSearchOptions & { sort?: string }): Promise<VrcSearchPage<VrcWorldSummary>>;
  searchGroups(query: string, o?): Promise<VrcSearchPage<VrcGroupSummary>>;
  searchAvatars(query: string, o?): Promise<VrcSearchPage<VrcAvatarSummary>>;
}

interface VrcLookupOptions { readonly cached?: boolean; readonly signal?: AbortSignal }
interface VrcSearchOptions { readonly offset?: number; readonly signal?: AbortSignal }
```

A lookup with no answer resolves `undefined`; unknown strings are `''` and unknown tri-states
`undefined`. `PERFORMANCE_RANKS` = `'Excellent' | 'Good' | 'Medium' | 'Poor' | 'VeryPoor'`, and
`rankIndex` turns one into 0–4 (`undefined` for `''`).

A picture's URL is usually VRCNext's local image cache, which only loads inside the app. Before
putting one in an embed, a webhook or anything else that leaves, pass it through
`publicImageUrl(url)` — it gives back `''` for an address only this machine can reach, so the field
drops rather than rendering blank. See [VRChat data](vrchat-data.md#pictures-that-leave-the-app).

Show a rank with `rankLabel(rank)` — `Very Poor`, not `VeryPoor`, and `Unknown` for `''` — and
`rankEmoji(rank)` for VRChat's traffic-light colour (🟢🔵🟡🟠🔴, ⚪ unknown); `RANK_EMOJI` is that
map. Never print a raw rank: the worst one is spelled as one word and reads as a typo.

`trustRank(tags)` reads a trust rank out of a user's tags and returns `{ label, short }` —
`Trusted User`, `Known User`, `User`, `New User` or `Visitor`. VRChat's tags are offset by one
from the label it shows, and this applies the same offset VRCNext's own profile badge does, so a
plugin never disagrees with the page about someone's rank.

## Locations

```ts
function parseLocation(location: string): ParsedLocation;
function isGroupInstance(type: string): boolean;
function instanceTypeLabel(type: string): string;

interface ParsedLocation {
  readonly worldId: string;      // '' when the location is not an instance
  readonly instanceId: string;
  readonly key: string;          // 'wrld_…:12345' — identifies the instance across visits
  readonly instanceType: InstanceType | '';
  readonly groupId: string;      // group instances
  readonly ownerId: string;      // friends / friends+ / hidden / private instances
  readonly region: string;
}
```

`INSTANCE_TYPES` = `'public' | 'friends+' | 'friends' | 'hidden' | 'private' | 'invite_plus' |
'group-public' | 'group-plus' | 'group-members'`, spelled as VRCNext spells them.

`instanceTypeLabel` gives the name the app puts on screen — "Public", "Friends+", "Group
Members" — from the same map its instance badges use, so an option list reads the way the rest
of VRCNext does. `INSTANCE_TYPE_LABELS` is that map. Use it rather than showing a raw type.

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
  /** The only fetch a plugin has. Asks about each host once; carries the plugin's abort signal. */
  fetch(url: string | URL, init?: RequestInit): Promise<Response>;
}
interface ClipboardApi {
  writeText(text: string): Promise<void>;
  readText(): Promise<string>;
}
```

### How `ctx.http.fetch` travels

While the [bridge](native-companion.md) is connected, every request goes through its `outbound`
service rather than the page. The page may only read a cross-origin response the server agreed
to share, so an API that sends no CORS headers, such as the Steam Web API, is unreachable from
the page. The bridge is not a browser and reaches it. When the bridge is not connected, the
request falls back to the page's own `fetch` and CORS applies again.

Through the bridge, a request has these limits:

| Limit | Value |
| :--- | :--- |
| Schemes | `http` and `https` |
| Methods | `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE` |
| Request body | a string, at most 1 MiB |
| Response body | UTF-8 text, at most 16 MiB after decompression |
| Headers | at most 32. `host`, `content-length`, `connection`, `keep-alive`, `proxy-connection`, `te`, `trailer`, `transfer-encoding`, `upgrade` and `user-agent` are refused as the connection's own |
| Credentials | **never leave this machine.** `Authorization`, `Proxy-Authorization`, `Cookie` and `Set-Cookie` are refused, a URL carrying `user:password@` is refused, and no request sends cookies — there is no cookie jar, and the page fallback is asked for `credentials: 'omit'`. Authenticate the way the API says to: a query parameter, or its own header such as `X-Api-Key` |
| Timeout | 30 s by default, 120 s at most |
| Redirects | never followed. A `3xx` comes back as it was sent, `Location` and all; follow it with another `ctx.http.fetch`, which asks about the new host |
| Addresses | public internet only. Loopback, private ranges, link-local (including `169.254.169.254`), CGNAT, multicast and the rest are refused, whether written in the URL or reached by resolving a name |

This machine holds two credentials a third party must never see — the bridge's pairing token,
which grants everything the bridge can do, and the page's VRChat session — so the rule is enforced
in three places rather than trusted: the host refuses the request before it is even put to the
user, the bridge refuses the header, and the finished request is checked once more on its way out.
`CREDENTIAL_HEADERS` and `isCredentialHeader(name)` are exported if you want to check first.

Aborting, through `init.signal` or by the plugin being disabled, rejects the promise with an
`AbortError` straight away. The bridge may still finish the request, but its answer is dropped.
`response.url` is the URL that was asked for.

## UI

```ts
interface UiApi {
  addNavTab(options: NavTabOptions): PanelHandle;
  addDashboardCard(options: DashboardCardOptions): PanelHandle;
  addSettingsCard(options: SettingsCardOptions): PanelHandle;   // options.section files it under your own section
  addSettingsSection(options: SettingsSectionOptions): SettingsSectionHandle;
  addSettingsDivider(): PanelHandle;
  addSidebarGroup(options: SidebarGroupOptions): PanelHandle;
  injectCss(css: string): PanelHandle;
  toast(options: ToastOptions): void;
  pickEntity(options: EntityPickOptions): Promise<readonly string[] | undefined>;   // the settings picker, on demand
  readonly kit: UiKit;
  createPanelLayout(): HTMLElement;
  createCard(title: string, icon: IconName): HTMLElement;
  createToggleRow(label: string, checked: boolean, onChange: (next: boolean) => void): HTMLElement;
}
```

```ts
interface PanelHandle extends Disposable {
  readonly element: HTMLElement;
  readonly visible: boolean;           // is the user looking at this panel?
}
interface NavTabOptions {
  readonly label: string;
  readonly icon: IconName;
  render(container: HTMLElement): void | Promise<void>;   // once, lazily, on first open
  readonly group?: string;
  onVisibility?(visible: boolean): void;                  // every switch, including to VRCNext's own tabs
}
interface SettingsSectionHandle extends PanelHandle {
  readonly sectionId: string;          // the data-section VRCNext switches on: "<plugin id>.<id>"
  readonly active: boolean;
  attach(block: HTMLElement): void;    // a card or container; removed with the section
  open(block?: HTMLElement): void;     // Settings tab → this section, scrolled to block
}
interface SidebarShortcut { id: string; label: string; icon: IconName; activate(): void }
```

`NavTabOptions` · `DashboardCardOptions` · `SettingsCardOptions` · `SettingsSectionOptions` ·
`SidebarGroupOptions` · `PanelHandle` · `IconName` · `ToastOptions` · `EntityPickOptions`

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
  textArea(options: { value: string; placeholder?: string; rows?: number; onCommit: (next: string) => void }): HTMLTextAreaElement;
  dropdown(options: { options: readonly { value: string; label: string }[]; selected: string; onChange: (next: string) => void }): HTMLSelectElement;
  slider(options: UiSliderOptions): HTMLElement;          // value readout, optional labelled markers
  chips(options: UiChipsOptions): HTMLElement;            // toggle buttons; the pressed ones are chosen
  timeField(options: UiTypedFieldOptions): HTMLInputElement;
  colorField(options: UiTypedFieldOptions): HTMLInputElement;
  listItem(options: UiListItemOptions): HTMLElement;      // VRCNext's compact profile row

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
