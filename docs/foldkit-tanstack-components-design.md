# Design Plan: TanStack-backed Components for Foldkit (pocket UI kit)

Drafted 2026-09-16. Scope: detailed design for six Foldkit component packages built on TanStack cores and foldkit-plus. Parent docs: [tanstack-foldkit-pocket.md](tanstack-foldkit-pocket.md), [react-in-foldkit.md](react-in-foldkit.md), [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md).

**Versions inspected (npm pack, type declarations):**

| Package | Version | Runtime dependencies | Notes |
|---|---|---|---|
| `@tanstack/table-core` | 9.2.4 | `@tanstack/store` | Extra entry points: `static-functions`, `reactivity`, `store-reactivity-bindings`, `experimental-worker-plugin` |
| `@tanstack/virtual-core` | 3.17.11 | none | |
| `@tanstack/hotkeys` | 0.8.0 | `@tanstack/store` | |
| `@tanstack/charts` | 0.18.0 | d3 submodules | |
| `@tanstack/ranger` | 0.0.4 | none | |
| `@tanstack/form-core` | 1.33.5 | `@tanstack/store`, `pacer-lite`, `devtools-event-client` | |

Foldkit `0.160.0`. foldkit-plus from its README only.

Legend: **[V]** verified in type declarations, **[U]** unverified assumption to confirm during the spike.

**Plan principles (updated):**
- **Vanilla and plus are equal.** Every component is complete with vanilla Foldkit (`core`) and native with foldkit-plus (`/plus`); see §0.5a.
- **TanStack is swappable.** TanStack libraries are the **default engines behind our own interfaces**, not the component contract; see §0.5b.
- **Our state.** State Schemas, Messages, codecs and public contracts are ours and engine-independent.
- **Every component is a Bundle.** Each package exports a `foldkit-bundle` Bundle (vanilla) with relative Surfaces/Mirrors/Slots via `foldkit-bundle-surface` (plus); see §0.2 and §0.8, and [foldkit-bundle-implementation-plan.md](foldkit-bundle-implementation-plan.md).

---

## 0. Shared foundations (all packages)

### 0.1 Package layout
```
packages/ui-foldkit/
  shared/        ids, Schema helpers, Mount utilities, Mixins slot/capability conventions, a11y helpers
  table/         @pocket/foldkit-table
  virtual/       @pocket/foldkit-virtual
  hotkeys/       @pocket/foldkit-hotkeys
  charts/        @pocket/foldkit-charts   (+ server renderer for email/PDF)
  range/         @pocket/foldkit-range
  forms/         @pocket/foldkit-forms
  examples/      one Foldkit app per package + kitchen sink
```
Each package: `Model` Schema, `Message` union, `update`, `view` (via `defineView`), `init`, Mounts/Subscriptions, Mixins `Slots`, optional foldkit-plus `Surface` helper, tests (`foldkit/test`, `Scene`), docs page, registry item.

### 0.2 Component shape: a Foldkit **Submodel**, packaged as a **Bundle**
> **Updated:** the parts below are the *contents* of each component's `Bundle.make({...})` (see §0.8). Hosts no longer hand-wire `foldChild`/`Subscription.lift`/`ManagedResource.lift`/`h.submodel`; they place the bundle.

Every component is a Submodel embedded in a host app:
```ts
// per component
export const Model: Schema.Struct<...>           // serializable, replayable (time travel)
export const Message: Schema.TaggedUnion<...>    // fact-named: SortedBy, VisibleRangeChanged…
export const init: (config) => Model
export const update: (model, message, config) => [Model, Commands]
export const view: SubmodelView<Model, Message>  // defineView
export const Slots                                // foldkit-mixins Slots.define
export type OutMessage                            // events the parent cares about (RowActivated, ValueCommitted…)
```
- **State ownership:** the component's Model is the single owner. TanStack instances are **derived, disposable computation**, rebuilt or synced from the Model, never the source of truth.
- **Parent integration:** the host maps child Messages (Foldkit Submodel pattern) and reacts to `OutMessage`s.
- **Config vs Model:** static config (columns, formatters, chart spec) lives outside Model (functions aren't serializable). Only data needed for replay/URL/sync goes in Model.

### 0.3 The TanStack bridge pattern
Three kinds of TanStack cores, three bridges:

| Kind | Cores | Bridge |
|---|---|---|
| **Pure compute** (no DOM) | table-core row models, charts scene/SVG, hotkeys parsing/formatting | Called in `view`/`update` from Model; memoized with `createLazy` keyed on inputs |
| **DOM-observing** | virtual-core (scroll/resize), ranger (pointer drag), charts interaction | **`Mount.defineStream`**: acquire instance on the element, translate callbacks into a `Stream<Message>`, release on unmount. [V] Mount shape `f(element, viewStateChanges) => Stream<Message>`, finalizers run on unmount |
| **Global listeners** | hotkeys manager | **Foldkit `Subscription`** (app-level) or Mount (element-scoped) |

**Rule:** TanStack callbacks may only *emit Messages*. They never mutate Model or call back into TanStack with derived state. The next render passes the updated Model back in.

### 0.4 Replay / time travel safety
Foldkit replays Messages and renders historical views with **no-op dispatch** for historical Mounts. Therefore:
- Mounts must be **idempotent** on re-acquire and tolerate no-op dispatch.
- Anything derived from DOM (measured sizes, scroll offset) that matters for replay is stored in Model as a fact (`ItemMeasured`, `ScrolledTo`).
- `viewStateChanges` (`Live`/`Paused`) [V]: pause DOM observers while `Paused` to avoid emitting during replay.

### 0.5 foldkit-plus integration conventions
- **Mixins:**
  - Every visual part is a named Slot with a Capability (`Container`, `Collection`…; extend with `Table`, `Row`, `Cell`, `Handle`, `Track`, `Field`) [U: custom capability definition API].
  - Default styles ship as a `Style` set built from pocket theme tokens.
  - Behaviors (hotkeys, drag) attach through Slots.
- **Mirror:** components expose **field references** for mirrorable state (table sort/filter/page, chart zoom range, range values) so hosts can `Mirror.url` / `Mirror.kv` them [U: field-ref API for Submodel fields].
- **Surface:** each package exports a `surface(App, path)` helper declaring a Projection (read-only view of component state) and a Message subset (safe actions), so hosts can expose them via **foldkit-agent** (MCP/WebMCP) in one line.
- **Remote:** table (server mode) and charts (server aggregates) read server-owned data through **foldkit-remote** queries; components accept data as input and never fetch.

### 0.5a Dual support: vanilla Foldkit and foldkit-plus as equals (supersedes 0.5 where they conflict)
**Requirement:** every component is fully usable with **plain Foldkit only**, and feels **native** under foldkit-plus. Neither is a degraded mode.

**Layering per package**
```
@pocket/foldkit-table
  ├─ core          (depends on: foldkit, effect, @tanstack/*)        ← vanilla-complete
  │    model.ts     Model Schema, Message union, init, update (pure)
  │    engine.ts    TanStack bridge (pure compute, state converters)
  │    parts.ts     Parts contract: named parts + default attributes per part
  │    view.ts      defineView(model, h, options) using Parts
  │    mounts.ts    Mount.defineStream definitions (drag, resize, scroll, hotkeys scope)
  │    codecs.ts    URL/KV Schema codecs for mirrorable state (usable by anyone)
  │    commands.ts  debounce/async Effects
  │    subscriptions.ts
  └─ plus          (subpath export "@pocket/foldkit-table/plus"; optional peer deps on foldkit-plus packages)
       slots.ts     Mixins Slots derived from Parts (same names, capabilities)
       behaviors.ts Mixins Behaviors wrapping the core Mounts
       surfaces.ts  Surface definitions (view / agent / readOnly / remote) + Module
       surface-view.ts SurfaceView.define(...) using the core view
       mirrors.ts   Mirror.url / Mirror.kv helpers using core codecs
       remote.ts    foldkit-remote query/mutation wiring for server mode
```
- `core` **never imports** foldkit-plus. `plus` imports only core public APIs, so no private coupling.
- foldkit-plus packages are **optional peer dependencies**; the `plus` subpath fails fast with a clear error if they're missing.

**The neutral seams (designed once, consumed by both)**
| Concern | Neutral core seam | Vanilla usage | foldkit-plus usage |
|---|---|---|---|
| Styling / customization | **Parts contract**: `parts: { root, headerCell, … }`, each `(ctx) => Attribute[]` with defaults; view calls `parts.x(ctx)` at the exact DOM position | Pass `parts` overrides (classes, inline styles, extra attributes, event handlers) as view options | `plus/slots.ts` maps each Part 1:1 to a Mixins Slot; `slots.x.attrs()` feeds the same position. `Style.attach` works |
| DOM interaction | **Mount definitions** exported from `mounts.ts` | Attach with `h.OnMount(Table.Mounts.ColumnResize(args))` (the default view already does) | Wrapped as **Behaviors** on the matching Slot; users can swap/extend behaviors |
| Public contract (observe / cause) | **Projection functions + Message subsets** as plain exported data: `projections.agent(model)`, `messageSets.agent = [SortingSet, …]` | Hosts build their own tool APIs, docs, or tests from these | `plus/surfaces.ts` builds Surfaces from the same functions/sets → foldkit-agent, Module.validate/toMermaid |
| URL / persistence | **Schema codecs** `codecs.url`, `codecs.kv` + `pick/apply` helpers for state slices | Host subscribes to URL changes and dispatches `SortingSet`/`PageIndexSet` via Foldkit `navigation`/`url`; writes URL in a Command | `plus/mirrors.ts` = `Mirror.url` / `Mirror.kv` using the same codecs |
| Server data | **Data-in / query-out**: `rows` input + `ServerQueryChanged` OutMessage + `toPocketQuery` | Host runs an Effect Command (HttpApiClient/Rpc) and feeds results back as a Message | `plus/remote.ts` wires the query to foldkit-remote (cache, optimistic, live updates) |
| Hotkeys | Registry `define` + Subscription + scoped Mount (core) | Use directly | Registry bound to a Surface → type-restricted Messages; Behavior attach |
| Forms drafts | `codecs.draft` + `DraftRestored` Message | Host persists with `KeyValueStore` Command | `Mirror.kv` |
| Replicated state | none in core (not a component concern) | n/a | foldkit-sync where a host wants it |

**Rules that keep parity honest**
1. **Every plus feature must have a core seam.** plus may *wrap* or *derive*, never add behavior that core lacks (except foldkit-plus-only capabilities like agents/sync/Module validation).
2. **Same Messages, same Model.** plus never introduces new state or reducers; a vanilla app and a plus app produce identical Message logs for identical interactions.
3. **Parts names = Slot names.** Renames happen in one place (`parts.ts`); plus derives Slots from it (type-level check).
4. **Defaults render identically.** Default Parts attributes and default Mixins Styles are generated from the same token-based style source.
5. **Docs show both.** Every page has "Foldkit" and "foldkit-plus" tabs; examples exist for both.

**Testing parity**
- A shared test suite runs each component in **two harnesses**: vanilla host and plus host (SurfaceView + Slots + Behaviors + Mirror).
- Assertions: identical Model after the same interaction script; identical DOM (modulo mixin-added classes); identical Message log.
- plus-only extras tested separately: `Module.validate(Project) == []`, agent tool schemas match message sets, Mirror round-trip equals core codec round-trip.
- CI matrix: `core` built/tested **without** foldkit-plus installed (ensures no accidental imports; also enforced with lint/`impound`).

**Versioning**
- core pins exact Foldkit (currently `0.160.0` with `effect@4.0.0-rc.115`).
- plus declares a tested foldkit-plus range; a compatibility table lives in docs.
- If foldkit-plus lags a Foldkit release, core ships anyway and plus follows.

**Impact on the component sections below**
- Wherever this doc says "Slots", read **Parts (core) → Slots (plus)**.
- "Surface/Agent" subsections describe `plus/surfaces.ts`, built on core **projections + message sets**.
- "Mirror" subsections describe core **codecs** first, plus `Mirror` helpers second.
- "foldkit-remote" in server mode is the plus path; the vanilla path is data-in/query-out with a host Command.
- Mount-based interactions are core; Behaviors are the plus wrapping.

### 0.5b Engine abstraction: TanStack is the default engine, not a hard dependency
Consistent with pocket's "not locked in" principle. **Only `engine-tanstack` modules import `@tanstack/*`.**

**Layering update (per package)**
```
core/
  engine.ts             Engine interface (pure TS types + Context.Service tag for Effect-side use)
  model.ts / view.ts …  depend ONLY on engine.ts types and our own Schemas
engines/
  tanstack.ts           default implementation (subpath export "@pocket/foldkit-<pkg>/engine-tanstack")
  <alt>.ts              optional alternatives (in-house, other libraries)
plus/                   unchanged (0.5a); never touches engines directly
```
- The engine is passed as **component config** (`Table.make({ engine: TanStackTableEngine, … })`): a plain object, because Foldkit views/updates are synchronous.
- The same engine is also exposed as an Effect **Layer** (`TableEngine.layerTanStack`) for server-side use (e.g. charts rendering, server-side table exports).
- **State Schemas are ours.** They intentionally mirror TanStack shapes today (`SortingState` etc.) for zero-cost conversion, but URL codecs, KV persistence, Surfaces/agent contracts and Message payloads never reference TanStack types. Engine adapters convert at the boundary.
- **Conformance test suite per engine interface.** Any engine must pass the same property tests (with Effect `Arbitrary`) and golden tests, so swaps are safe.
- **Bundle hygiene:** engines import only what they use (table-core v9 opt-in features and row models; charts marks as needed); charts engine lazy-loaded.

**Engine interfaces (sketch)**
| Package | Interface (pure unless noted) | Default engine | Realistic alternatives |
|---|---|---|---|
| table | `rows(state, data, columns, features) → RowModel`; `headerGroups(state, columns) → HeaderGroup[]`; `nextSorting(state, columnId, multi)`; `nextSelection(state, rowId, mode)`; `columnOffsets(state, columns)` (pinning/sizing); `facets(data, columnId)` | `@tanstack/table-core` 9.x (mature; keep) | In-house minimal (sort/filter/paginate only) for tiny bundles; server-only engine (no client row models; server mode delegates entirely to pocket) |
| virtual | `observe(element, options) → { windows: Stream<VirtualWindow>, measure(el, key), scrollTo(req), update(options), dispose }` (DOM, used inside Mount) | `@tanstack/virtual-core` 3.x (mature; keep) | In-house fixed-size virtualizer; native CSS `content-visibility` fallback engine |
| hotkeys | `parse(keys)`, `normalize(keys)`, `matches(event, parsed)`, `format(parsed, platform)`, `register(target, bindings, emit) → dispose` (DOM), `record(element, emit)` (DOM) | `@tanstack/hotkeys` 0.8 (early) | **In-house matcher** (small; likely worth building as fallback) |
| charts | `scene(spec, data, size, theme) → Scene`; `toSvg(scene) → string`; `sceneNodes(scene) → Node tree` (for Foldkit walker); `nearest(scene, x, y) → Point \| null` | `@tanstack/charts` 0.18 (young; d3-heavy) | Observable Plot (server SVG); hand-built sparkline/KPI engine for emails |
| range | `valueForPosition(clientX, rect, config)`, `snap(value, config)`, `nextStep(value, direction, config)`, `ticks(config)`, `segments(values, config)`, `drag(element, handles, config, emit) → dispose` (DOM) | `@tanstack/ranger` 0.0.4 (very early) | **In-house engine as primary candidate** (math is small; keyboard logic already pure) |
| forms | `validate(schema, values) → FieldErrors` (default: Effect Schema engine); optional `FormEngine` for linked fields/arrays | **Effect Schema engine** (ours) | `@tanstack/form-core` controlled adapter |

**Decision points (per milestone 1 spikes)**
- **range:** build the in-house engine alongside the ranger engine; ship in-house as default if parity tests pass with less code. ranger stays as an optional engine.
- **hotkeys:** start with TanStack engine; build an in-house matcher when scoped hotkeys land (M2) and keep both behind conformance tests.
- **charts:** TanStack engine default; evaluate Plot for server-only rendering if bundle/runtime issues appear (d3-selection side effects on server).
- **table / virtual:** TanStack engines are the long-term default; abstraction protects against major-version churn (v8→v9 was a large break).

**Impact on sections below**
- References to TanStack APIs inside each component section describe the **TanStack engine implementation**, not the component contract.
- Component Models, Messages, Parts/Slots, codecs, Surfaces and tests are engine-independent; engine-specific tests live under `engines/`.
- Milestone 1 of each component now includes: define the engine interface, implement the TanStack engine, write the conformance suite.

### 0.8 Packaging every component as a Bundle
Built on `foldkit-bundle` / `foldkit-bundle-surface` ([implementation plan](foldkit-bundle-implementation-plan.md)). The bundle packages formalize the per-component shape this doc designed by hand.

**Mapping of this doc's concepts to bundle concepts**
| This doc | Bundle equivalent |
|---|---|
| Model / Message / init / update / OutMessage (§0.2) | `Bundle.make({ Model, Message, init, update })`, OutMessage inferred |
| Static config (columns, spec, formatters) | Bundle **args** (`at(link, args)`); args threaded to `update(model, message, args)` |
| Engine (§0.5b) | An arg: `Table.bundle({ engine: TanStackTableEngine, … })` |
| Subscriptions (hotkeys global registrations, resize for charts) | `subscriptions(args)`, lifted, gated and namespaced by placement |
| Mounts (virtual scroll, range drag, column resize) | Stay in the bundle's `view` (Mounts attach in views); `/plus` wraps them as Behaviors |
| Programmatic entry points (`scrollToIndex`, `setSorting`, `reset`) | Bundle `helpers` → lifted to parent `Step`s |
| Parts contract (§0.5a) | Bundle `view` publishes Parts; `/plus` derives Slots |
| Projections + message sets (§0.5a) | Bundle **relative Surfaces** (bundle-surface), re-rooted per placement |
| URL/KV codecs (§0.5a) | Bundle **mirrorable fields** → `placed.mirrors.url(prefix)` / `.kv(key)` |
| Vanilla host wiring (§0.5a) | `Bundle.place` / `.at` + `Bundle.assemble` |
| foldkit-plus host wiring | same placement + `placed.surfaces`, `placed.mirrors`, `placed.contract` in `Module` |

**Core/plus split under bundles** (replaces the per-package `core/`/`plus/` folders' wiring code):
```
@pocket/foldkit-table
  core/   bundle.ts (Bundle.make), model.ts, engine.ts, view.ts (Parts), mounts.ts, commands.ts, codecs.ts
  plus/   surfaces.ts (relative Surfaces), mirrors.ts (mirrorable fields), slots.ts (Parts → Slots)   ← consumed by foldkit-bundle-surface
  engines/tanstack.ts
```
Parity rule (§0.5a) becomes: **the vanilla bundle and the plus-enriched placement produce identical Message logs**; plus only adds derived declarations.

**Per-component bundle shape**
| Component | Model (owned) | Args | Subscriptions / resources | View + Mounts | Helpers | OutMessages | Typical placement |
|---|---|---|---|---|---|---|---|
| **Table** | sorting, filters, globalFilter, pagination, selection, visibility, order, pinning, sizing, expanded, grouping | columns, getRowId, features, mode, engine, strings, virtualize? | none (server data arrives via host/Remote) | table Parts; column-resize Mount; nests Virtual/Range/Hotkeys | `setSorting`, `setFilter`, `clearFilters`, `setPage`, `selectRows`, `reflectState` | `RowActivated`, `SelectionChanged`, `ServerQueryChanged` | `Bundle.place("invoicesTable", InvoicesTable)`; row groups via `each` |
| **Virtual** | window (range, items, totalSize), measurements, pendingScrollTo | count source, estimateSize, overscan, horizontal, lanes, followOnAppend, engine | none (DOM work is Mount-scoped) | viewport/spacer/item Parts; scroller Mount; item measure Mount | `scrollToIndex`, `scrollToOffset`, `setCount` | `EndReached` | Inside Table via `Link.compose`; standalone for logs/feeds |
| **Hotkeys** | overrides, enabled groups, recording state | registry (`Hotkeys.define`), platform, engine | **global registrations as subscriptions** (gated by enabled flags; lifted automatically) | help overlay view; recorder Mount; scoped shortcuts = Behavior/Mount attached by *other* bundles' views | `rebind`, `resetBinding`, `startRecording`, `enableGroup` | the registry's mapped Messages (routed to host), `BindingRecorded` | One app-level placement; scoped registries inside Table/Forms bundles |
| **Charts** | size, focus, zoom, hiddenSeries | spec, theme, engine, data selector | none (resize via Mount) | SVG (static or scene-walker); resize + pointer Mounts | `resetZoom`, `toggleSeries`, `focusPoint` | `PointActivated` | One per widget; dashboard grid via `each` (key = widget id) |
| **Range** | values, dragValues, activeHandle, focusedHandle | min, max, step/steps, minDistance, ticks, format, interpolator, engine | none | track/handle/tick Parts; drag Mount; keyboard via `h.OnKeyDown` (pure) | `setValues`, `reflectValues` | `ValueCommitted` | Inside Table filters and Forms fields via `Link.compose`; standalone |
| **Forms** | values (encoded), touched, dirty, errors, asyncValidation, submit, submitCount | schema, fields spec, validateOn, submit command factory, strings | optional: draft autosave as subscription (vanilla) / Mirror.kv (plus) | field Parts; nested Range/Listbox/Upload bundles | `reset`, `setValues`, `restoreDraft`, `submit` | `Submitted`, `SubmitFailed`, `Cancelled` | One per form; repeatable groups via `each`; relation fields place a Listbox (`fromParts`) |

**Nesting (composition via `Link.compose` / `each`)**
```
RecordsBrowser (host)
 ├─ Table            Bundle.place("records", RecordsTable)
 │   ├─ Virtual      Link.compose(recordsLink, field("rows"))           rows viewport
 │   ├─ Range        each(filters, key = columnId)                       numeric/date range filters
 │   └─ (scoped hotkeys Behavior on table Parts)
 ├─ Forms            Bundle.place("editor", RecordForm)
 │   ├─ Listbox      Link.compose(editorLink, field("relation"))        @foldkit/ui via fromParts
 │   └─ Group        each(field("lineItems"), key = itemId)             repeatable field groups
 ├─ Charts           each("widgets", key = widgetId)
 └─ Hotkeys          Bundle.place("shortcuts", AppHotkeys)
```

**What bundles fix in this plan**
1. **One placement per use.** Replaces 4–5 hand lifts; forgetting a lift is a compile error (bundle Phase 2).
2. **Nesting is first-class.** Table ⊃ Virtual ⊃ Range filters and Forms ⊃ Listbox ⊃ field groups via `Link.compose`/`each`.
3. **Many instances without collisions.** Namespaced subscription keys; per-placement URL params (`?invoices.sort=…`, `?orders.sort=…`); per-placement agent tools.
4. **Declarative admin assembly.** The pocket admin page is one `Bundle.assemble([...])`; `Module.validate` checks ownership; `Module.toMermaid` diagrams the tree.

**What bundles don't change**
- **Engine bridges stay our work:** table state sync, Virtualizer lifecycle, ranger math, chart scenes.
- **Element interactions stay Mounts/Behaviors** attached in bundle views.
- **Server-side chart rendering** stays a plain Effect `Charts` service.
- **TanStack Query/DB stay unused** in Foldkit (foldkit-remote/sync own those roles).
- **Per-item *resources* in `each`** remain an open bundle design point; none of these six components needs them.

### 0.6 Accessibility & i18n baseline
- ARIA patterns: grid/table (`role=grid` optional), listbox virtualization (`aria-setsize`/`aria-posinset`), slider (`role=slider`, `aria-valuenow/min/max`, keyboard), charts (`role=img` + accessible data table fallback), forms (labels, `aria-invalid`, `aria-describedby` errors).
- All user-visible strings via a `Messages` dictionary config; numbers/dates via `Intl`.

### 0.7 Testing strategy (all packages)
- `update` unit tests (pure): Message → Model.
- Scene tests (`foldkit/test`) for view output and event → Message.
- Property tests with Effect `Arbitrary`: e.g. table controlled state ⇄ URL Mirror round-trip; range value invariants.
- Browser tests (vitest browser / Playwright) for Mount behavior (scroll, drag, resize, hotkeys).
- Replay tests: record Messages, replay, assert identical Model and DOM.

---

## 1. `@pocket/foldkit-table`: table-core as a controlled engine

### 1.1 Goals
- **Features:** sorting, column filters, global filter, pagination, column visibility/order/pinning/sizing, row selection, expansion, grouping. Opt-in per table.
- **Modes:** client mode (rows in memory) and **server mode** (state becomes the pocket query).
- **Integrations:** state mirrorable to URL; fully restylable via Mixins; actions agent-operable via Surface.
- **Composition:** works with `@pocket/foldkit-virtual` for large row counts.

### 1.2 Relevant table-core v9 facts
- **Construction:** `constructTable(tableOptions)` [V]. `tableFeatures`, `stockFeatures`, `coreFeatures` [V]. **Features are opt-in**, e.g. `rowSortingFeature`, `columnFilteringFeature`, `rowPaginationFeature`, `rowSelectionFeature`, `columnPinningFeature`, `columnSizingFeature`, `columnVisibilityFeature`, `columnOrderingFeature`, `rowExpandingFeature`, `columnGroupingFeature`, `globalFilteringFeature`, `cellSelectionFeature` [V].
- **Row model factories:** `createCoreRowModel`, `createSortedRowModel`, `createFilteredRowModel`, `createPaginatedRowModel`, `createGroupedRowModel`, `createExpandedRowModel`, `createFacetedRowModel` / `UniqueValues` / `MinMaxValues` [V].
- **Built-in functions:** `sortFn_*`, `filterFn_*`, `aggregationFn_*` [V].
- **State options [V]:**
  - `initialState`, `data`, `key` (devtools)
  - `atoms?: ExternalAtoms`: "your own external writable atoms for individual state slices… takes precedence over `options.state[key]`"
  - `options.state` also exists
  - Types: `SortingState`, `ColumnFiltersState`, `PaginationState`, `RowSelectionState`, `ExpandedState`, `ColumnPinningState`, `ColumnSizingState`, `ColumnVisibilityState`, `ColumnOrderState`, `GroupingState` [V]
- **Helpers:** `functionalUpdate`, `makeStateUpdater`, `Updater`, `OnChangeFn` [V]. `static-functions` entry point [V] (likely API without prototype instances [U]). `flex-render` entry for cell templates [V].

### 1.3 State design
```ts
// Model (serializable)
TableModel = Schema.Struct({
  sorting:          Schema.Array(Schema.Struct({ id: Schema.String, desc: Schema.Boolean })),
  columnFilters:    Schema.Array(Schema.Struct({ id: Schema.String, value: FilterValue })),
  globalFilter:     Schema.String,
  pagination:       Schema.Struct({ pageIndex: Schema.Int, pageSize: Schema.Int }),
  rowSelection:     Schema.Record(Schema.String, Schema.Boolean),
  columnVisibility: Schema.Record(Schema.String, Schema.Boolean),
  columnOrder:      Schema.Array(Schema.String),
  columnPinning:    Schema.Struct({ left: Schema.Array(Schema.String), right: Schema.Array(Schema.String) }),
  columnSizing:     Schema.Record(Schema.String, Schema.Number),
  expanded:         Schema.Union([Schema.Literal(true), Schema.Record(Schema.String, Schema.Boolean)]),
  grouping:         Schema.Array(Schema.String),
  resizing:         Schema.Option(ColumnResizeInfo),   // transient
})
```
- Shapes mirror table-core state types so conversion is identity. Schema codecs guarantee URL/KV decode safety.
- `FilterValue` is a Schema union (string | number range | date range | string[] | boolean), defined per column.

### 1.4 Controlled-engine bridge
Choose one after the spike:
- **A. `state` passthrough (preferred for purity).** Each render: `constructTable({ features, data, columns, state: model, … })`, memoized by `createLazy` on `(data, columns, model slices)`. All mutations go through Messages. Table API calls that would mutate state (`column.toggleSorting()`) are **not used**; header clicks dispatch `SortToggled({ id, multi })` and `update` applies the same semantics (reuse `functionalUpdate` + table-core's sorting cycle helpers where exposed [U]).
- **B. External atoms bridge.** Create per-slice writable atoms (`@tanstack/store` atoms) whose `set` emits a Foldkit Message instead of storing. `get` reads the current Model. Lets us call table-core's own mutation APIs (`toggleSorting`, `setPageIndex`) and keep exact semantics, while Model stays the owner. Needs a per-render "current Model" reference and care with synchronous reads-after-write [U: ExternalAtoms type contract].

**Decision criterion:** if table-core exposes pure "next state" helpers for each feature, use A; otherwise B, to avoid re-implementing multi-sort, selection ranges, pinning semantics.

### 1.5 Messages
```
SortToggled { columnId, multi }             SortingSet { sorting }
ColumnFilterSet { columnId, value }         GlobalFilterSet { value }      FiltersCleared
PageIndexSet { pageIndex }                  PageSizeSet { pageSize }
RowSelectionToggled { rowId, value? }       AllRowsSelectionToggled { value? }   RangeSelected { fromRowId, toRowId }
ColumnVisibilityToggled { columnId }        ColumnOrderSet { order }       ColumnPinned { columnId, side }
ColumnResizeStarted { columnId, startX }    ColumnResizeMoved { deltaX }   ColumnResizeEnded
RowExpandedToggled { rowId }                GroupingSet { grouping }
RowActivated { rowId }            → OutMessage (open record)
SelectionChanged { rowIds }       → OutMessage
ServerQueryChanged { query }      → OutMessage (server mode)
```
- Debounced filters: text input changes emit `ColumnFilterDraftChanged`; a Command (Effect `Effect.sleep` / Stream debounce) commits `ColumnFilterSet`. Drafts stay local, not mirrored.
- `autoReset*` semantics (reset page on filter change) implemented explicitly in `update` for predictability.

### 1.6 Config (non-serializable)
```ts
TableConfig<Row> = {
  id: string
  getRowId: (row) => string
  columns: ReadonlyArray<ColumnSpec<Row>>   // accessor, header, cell renderer (Foldkit view fn), filter kind, sortFn, enable* flags, meta
  features: { sorting?, filtering?, pagination?, selection?, pinning?, sizing?, visibility?, ordering?, expanding?, grouping? }
  mode: "client" | "server"
  pageSizes: number[]
  messages: TableStrings
  virtualize?: { estimateRowHeight: number; overscan?: number }   // uses @pocket/foldkit-virtual
}
```
**Column specs from pocket collections:** `Pocket.tableColumns(collectionSpec)` derives accessors, header labels (Schema `title` annotations), cell renderers per field type (date, relation chip, file thumb, boolean, JSON), filter kinds and sortability (indexed fields).

### 1.7 View & Slots
Slots (Mixins):
- **Frame:** `root`, `toolbar`, `globalFilter`, `table`, `thead`, `headerRow`, `headerCell`, `sortIndicator`, `resizeHandle`, `filterRow`, `filterCell`
- **Body and cells:** `tbody`, `row`, `cell`, `selectCell`, `expandToggle`, `groupRow`, `emptyState`, `loadingState`
- **Footer:** `pagination`, `pageButton`, `pageSizeSelect`, `columnMenu`

Semantics:
- Semantic `<table>` by default; `role="grid"` mode when cell navigation is enabled (arrow keys via `@pocket/foldkit-hotkeys` Behavior on `table` slot).
- Pinned columns: sticky positioning via computed offsets (table-core pinning/sizing APIs) emitted as inline style on `headerCell`/`cell` Slots.
- **Cell renderers** are Foldkit view functions `(cellContext, h) => Html` restricted to the table's Message universe plus `OutMessage`s.

### 1.8 Mirror (URL/KV)
- **URL:** `sorting`, `columnFilters`, `globalFilter`, `pagination` (**linkable views**).
- **KV:** `columnVisibility`, `columnOrder`, `columnSizing`, `columnPinning` (**per-device layout prefs**).
- **Transient, never mirrored:** `rowSelection`, `expanded`, `resizing`.
- **Compact URL codec** (Schema transformation), e.g. `?sort=created.desc,title&f.status=paid&q=acme&page=2&size=50`; unknown/invalid params are dropped via Schema decode (no crash).
- The package exports `TableMirrors.url(App, tablePath)` and `TableMirrors.kv(App, tablePath, key)` [U: Mirror API for nested fields].

### 1.9 Server mode (pocket)
- table-core runs with manual modes (`manualSorting/manualFiltering/manualPagination`-equivalent options in v9 [U]); rows = current server page.
- **`toPocketQuery(tableModel, columns) → RecordsQuery`:**
  - `sort=-created,title`
  - `filter` built with pocket's filter language: column filter kinds map to expressions (`status = "paid" && total >= 100 && created >= "2026-01-01"`), with values escaped.
  - `page`/`perPage`
  - `fields` from visible columns (projection)
  - `expand` from relation columns
- **Data flow:**
  1. State Message
  2. `update` emits `ServerQueryChanged`
  3. The host issues a **foldkit-remote** query keyed by `RecordsQuery`
  4. Remote supplies `{ items, totalItems }` into Model / host
  5. The table renders
- Loading/stale states come from Remote (keep previous page while fetching).
- **Realtime:** Remote live updates patch rows in place; if the change affects filters/sort, show a "new results" pill rather than jumping.
- **Faceting in server mode:** optional facet endpoint (pocket aggregates) feeding filter dropdowns.

### 1.10 Surface / Agent
```ts
TableSurface(App, path, { name: "invoices" })
// Projection: { visibleColumns, sorting, filters, pagination, totalItems, selectedRowIds, pageRows (id + summary fields) }
// Messages exposed: SortingSet, ColumnFilterSet, GlobalFilterSet, PageIndexSet, RowSelectionToggled, FiltersCleared
// + host-provided bulk actions (DeleteSelected, ExportSelected) with authorize via pocket rules
```
Agents manipulate the *same* table the user sees ("show unpaid invoices over $1k sorted by due date"). Bulk mutations go through host Messages and pocket rules.

### 1.11 Performance
- Memoize the table instance per `(data ref, columns ref, state slice refs)`.
- Client mode above ~10k rows: enable virtualization.
- Consider table-core `experimental-worker-plugin` [V exists] for sorting/filtering off the main thread later.
- Column resize: `ColumnResizeMoved` on pointer move is high-frequency. Use a Mount that batches with `requestAnimationFrame` and emits one Message per frame.

### 1.12 Milestones
1. **Spike (A vs B bridge):** sorting + pagination, client mode, Scene tests.
2. **Core features:** filters, global filter, selection, visibility; Slots + default Styles; URL/KV mirrors.
3. **Server mode:** `toPocketQuery`, Remote integration, realtime patching.
4. **Advanced:** pinning, sizing/resize, ordering (drag), expansion, grouping, keyboard grid navigation.
5. **Pocket integration:** collection-derived columns, Surface/Agent, virtualization, registry item, docs.

---

## 2. `@pocket/foldkit-virtual`: virtual-core via Mount

### 2.1 Goals
- **Virtualization:** vertical lists, horizontal lists, grids (lanes), dynamic item sizes, window scrolling.
- **List behaviors:** infinite loading, "stick to bottom" logs/chat, scroll-to-index.
- **Composition:** used by table (rows), logs viewer, realtime feeds, select dropdowns.

### 2.2 Relevant virtual-core facts [V]
- **Class:** `Virtualizer<TScrollElement, TItemElement>`.
- **Core options:** `count`, `getScrollElement`, `estimateSize(index)`, `scrollToFn`, `observeElementRect`, `observeElementOffset`, `onChange(instance, sync)`, `measureElement(el, entry, instance)`.
- **Layout options:** `overscan`, `horizontal`, `paddingStart/End`, `scrollPaddingStart/End`, `scrollMargin`, `gap`, `lanes`, `laneAssignmentMode`, `isRtl`.
- **Behavior options:** `initialOffset`, `getItemKey`, `rangeExtractor`, `indexAttribute`, `initialMeasurementsCache`, `anchorTo`, `followOnAppend`, `scrollEndThreshold`, `useScrollendEvent`, `enabled`, `useCachedMeasurements`.
- **Helpers:** `observeElementRect`, `observeWindowRect`, `observeElementOffset`, `observeWindowOffset`, `measureElement`, `elementScroll`, `windowScroll`, `defaultRangeExtractor`, `defaultKeyExtractor`.

### 2.3 State design
```ts
VirtualModel = Schema.Struct({
  count: Schema.Int,
  range: Schema.Struct({ startIndex: Schema.Int, endIndex: Schema.Int }),
  items: Schema.Array(Schema.Struct({ index: Schema.Int, key: Schema.String, start: Schema.Number, size: Schema.Number, lane: Schema.Int })),
  totalSize: Schema.Number,
  scrollOffset: Schema.Number,
  isScrolling: Schema.Boolean,
  atEnd: Schema.Boolean,                      // for infinite loading / follow
  measurements: Schema.Record(Schema.String, Schema.Number),   // key → measured size (replay-safe)
  pendingScrollTo: Schema.Option(ScrollRequest),               // command-like request consumed by Mount
})
```
Storing `items` (virtual items) in Model makes views pure and replayable. It's small (visible window only).

### 2.4 Mount design
```
Mount.defineStream("VirtualScroller", { id, count, estimateSize, overscan, horizontal, lanes, followOnAppend, … serializable options })
  acquire:
    virtualizer = new Virtualizer({
      ...options,
      getScrollElement: () => element,
      scrollToFn: elementScroll, observeElementRect, observeElementOffset,   // or window variants
      measureElement,
      onChange: (inst, sync) => emit(VirtualWindowChanged(snapshot(inst))),   // throttled to rAF
    })
    cleanup = virtualizer._didMount()   [U: mount/didMount API names in 3.17]
    virtualizer._willUpdate()           [U]
  on option change: virtualizer.setOptions(...) [U]; respect viewStateChanges Paused → stop emitting
  release: cleanup()
```
- **Item measurement:** each rendered item gets `data-index` (the `indexAttribute`) and an `ItemMeasure` Mount (or a single ResizeObserver in the scroller Mount observing children by attribute) → `ItemMeasured({ key, size })`, batched per frame.
- **Scroll requests:** `update` sets `pendingScrollTo`. The Mount watches Model-provided args [U: how Mount args updates are delivered], calls `virtualizer.scrollToIndex/Offset`, then emits `ScrollRequestConsumed`.
- **Non-DOM environments** (tests, SSR): Mount doesn't fire; `init` computes an initial window from `estimateSize` × viewport estimate so the view still renders something.

### 2.5 Messages
```
VirtualWindowChanged { range, items, totalSize, scrollOffset, isScrolling, atEnd }
ItemMeasured { key, size }         ItemsMeasured { entries }   (batched)
ScrollToIndexRequested { index, align, behavior }  ScrollToOffsetRequested { offset }
ScrollRequestConsumed
CountChanged { count }
EndReached → OutMessage (infinite loading: host loads next page via foldkit-remote)
```

### 2.6 View
- **Structure:** Slots `viewport` (scroll element, Mount attached), `spacer` (height = totalSize), `item` (absolutely positioned via `transform: translateY(start)`), `loadingRow`, `empty`.
- **Render function:** `renderItem(index, virtualItem, h)` supplied by config (Foldkit view fn).
- **A11y:** `aria-rowcount`/`aria-setsize`, `aria-posinset` on items; preserve focus when items recycle (keep focused key rendered via `rangeExtractor` including focused index).

### 2.7 Table integration
- The table view composes `VirtualList` for `tbody` rows (`display: grid` table layout, or a spacer-row technique for semantic tables).
- Sticky header and pinned columns remain in table Slots; the virtual viewport is the table's scroll container.

### 2.8 Milestones
1. Fixed-size vertical list, Mount + window snapshot, Scene + browser tests.
2. Dynamic measurement + scroll-to-index + replay safety.
3. Window scrolling, horizontal, lanes/grid.
4. Follow-on-append (logs/chat), infinite loading OutMessage.
5. Table integration; pocket logs viewer and realtime feed.

---

## 3. `@pocket/foldkit-hotkeys`: global Subscription or scoped Behavior

### 3.1 Goals
- **Kinds:** global app shortcuts (command palette `Mod+K`, save `Mod+S`), sequences (`g i` go to inbox), scoped shortcuts (active within table/editor area).
- **Discoverability:** declarative, typed registry that powers a help overlay and agent discovery.
- **Platform display:** `⌘K` vs `Ctrl+K`; recording custom bindings; conflict detection.

### 3.2 Relevant hotkeys facts [V]
- **Manager:** `HotkeyManager.register(hotkey, callback, options?) → HotkeyRegistrationHandle` (updatable `callback`, `setOptions`, `unregister`); `getHotkeyManager()`, `getSequenceManager()`, `getKeyStateTracker()`.
- **Options:** `conflictBehavior`, `enabled`, `eventType` (`keydown`/`keyup`), `ignoreInputs` (smart defaults), `platform`, `preventDefault`, `requireReset`, `stopPropagation`.
- **Parsing and formatting:** `parseHotkey`, `normalizeHotkey`, `validateHotkey`, `assertValidHotkey`, `matchesKeyboardEvent`, `parseKeyboardEvent`, `formatForDisplay`, `formatWithLabels`, `formatHotkeySequence`, `detectPlatform`, `MAC_MODIFIER_SYMBOLS`.
- **Matchers and recorders:** `createHotkeyHandler`, `createMultiHotkeyHandler`, `createSequenceMatcher`; `HotkeyRecorder`, `HotkeySequenceRecorder`.
- `@tanstack/store`-based state (manager internals only; not exposed to Model).

### 3.3 Registry design
```ts
const AppHotkeys = Hotkeys.define({
  openPalette: { keys: "Mod+K", description: "Open command palette", group: "General", message: () => Message.PaletteOpened() },
  save:        { keys: "Mod+S", description: "Save", when: (model) => model.dirty, message: () => Message.SaveRequested() },
  goInbox:     { sequence: ["G", "I"], description: "Go to inbox", message: () => Message.NavigatedTo({ route: "inbox" }) },
})
```
- **Typed:** `message` returns the app's Message type. `when` is a pure Model predicate (evaluated at dispatch time against the current Model [U: access to current Model from Subscription]; otherwise enabled flags are pushed via Subscription args).
- **Validation:** `validateHotkey` at define time (dev assertion), plus conflict detection across registries using `normalizeHotkey`.
- **User overrides:** stored in Model and mirrored to KV (`Mirror.kv`), applied on top of defaults.

### 3.4 Global: Foldkit Subscription
```
Subscription "Hotkeys.global"(activeBindings: derived from Model: enabled ids + overrides)
  acquire: for each binding → manager.register(keys, () => emit(binding.message()), { preventDefault, ignoreInputs, requireReset })
           sequences → getSequenceManager().register(...) [U: sequence register API]
  on args change: handle.setOptions / re-register changed ones
  release: handle.unregister() for all
```
- Subscriptions are keyed by `activeBindings` so enabling/disabling via Model re-syncs registrations [U: Foldkit Subscription dependency/keying semantics].
- Paused view state (time travel): disable all (`enabled: false`).

### 3.5 Scoped: Mixins Behavior
```ts
const TableKeys = Hotkeys.behavior(TableSlots.root, {
  up: { keys: "ArrowUp", message: () => TableMessage.FocusMoved({ dir: "up" }) },
  selectAll: { keys: "Mod+A", message: () => TableMessage.AllRowsSelectionToggled({}) },
})
TableView.pipe(Behavior.attach(TableKeys))   // [U: exact Behavior attach API]
```
- **Registration:** the Behavior contributes a Mount on the slot element registering handlers with a **target/scope** (manager options for target element [U]), or a local `keydown` listener using `createMultiHotkeyHandler` + `matchesKeyboardEvent` (no global manager; robust fallback).
- **Scope:** active only while focus is within the element (`focusin`/`focusout` tracked in Mount).
- **Type safety:** the Behavior is typed to the Surface/Slot's Message subset, so a scoped hotkey can't dispatch Messages that view isn't allowed to cause (foldkit-plus Surface guarantee).

### 3.6 Help overlay & recorder
- `Hotkeys.helpView(registries, h)` renders grouped shortcuts with `formatForDisplay` per platform.
- **Rebinding UI:** `HotkeyRecorder` in a Mount → `BindingRecorded({ id, keys })`; conflicts shown before commit.
- **Agent/Surface:** hotkey ids map to the same Messages, so the help overlay and agent tool list derive from one registry.

### 3.7 Milestones
1. Global Subscription with `define` registry, display formatting, tests with synthetic KeyboardEvents.
2. Scoped Behavior (local handler fallback), focus scoping.
3. Sequences, overrides + KV mirror, conflict detection, recorder UI.
4. Help overlay, command palette integration, pocket admin shortcuts.

---

## 4. `@pocket/foldkit-charts`: pure SVG, client + server

### 4.1 Goals
- **Charts:** dashboards (records over time, revenue, job throughput, latency percentiles), a Plot-like grammar.
- **Rendering:** pure SVG from Model data, SSR/hydration friendly; identical charts server-rendered for emails (stats digests) and PDFs (reports/invoices).
- **Interaction:** tooltips, crosshair, brush/zoom, legend toggles, driven by Messages.

### 4.2 Relevant charts facts [V]
- **Definition and rendering:** `defineChart(spec | responsive config)`, `createChartScene(definition, size, layout?) → ChartScene`, **`renderChartSvg(scene, options) → string`**, `renderChartSvgWithHooks`, `renderSceneNodes`.
- **Marks:** `lineX/Y`, `areaY`, `barX/Y`, `dot`, `rect/cell`, `hexagon`, `link`, `arrow`, `boxX/Y`, `ridgelineX/Y`, `differenceX/Y`, `linearRegression*`, `crosshair`, `frame`, `colorLegend`, `facet/facetChart`, `stack`, `group`.
- **Transforms:** `binX/Y/XY`, `binTimeX/Y`, `groupBy`, `rollingWindow`, `normalize`, `cumulative`, `dodgeX/Y`.
- **Interaction:** `findNearestPoint(scene, x, y)`, `viewportInteractionPoints`, `resolveFocusScene`, `focusedSceneNodes`, `whenFocused`.
- **Hosts:** `mountChart(container, options)` (DOM host), `mountChartRenderer`, `createChartRuntime`, `defineChartElement` (Lit custom element), adapters (`./adapter`, `./alpine`, `./angular`, `./canvas`…).
- **Theme:** `defaultChartTheme`.
- Depends on d3 submodules (scale, shape, array, brush, zoom…).

### 4.3 Architecture: two render paths, one spec
```
ChartSpec (config fn: data → defineChart(...))  +  ChartModel (Schema: data ref, size, focus, zoom, hidden series)
        │
        ├── Client (Foldkit):  createChartScene(def, size) → scene → SVG
        │       Option 1 (default): renderChartSvg(scene, opts) → string → insert as SVG markup [U: Foldkit raw SVG/HTML insertion]
        │       Option 2: scene nodes → Foldkit SVG builder (h.svg/g/path/rect/text) via renderSceneNodes-equivalent walker (native VDOM, diffable, event-bindable)
        │       Interaction Mount: pointer → findNearestPoint(scene, x, y) → Message.ChartFocused({ seriesId, index })
        │
        └── Server (Effect):   Charts.renderSvg(spec, data, size, theme) → Effect<string>  (pure; no DOM)
                → EmailTemplates (inline SVG or rasterize to PNG for email clients) / Documents (PDF)
```
- **Prefer Option 2 for interactive dashboards:** native Foldkit SVG nodes diff efficiently, allow `h.OnClick` on marks and Mixins Styles on `mark` Slots. **Option 1** for static and server output (exact parity with `renderChartSvg`).
- Pin the walker to the scene node types (`SceneNode`, `SceneGroup` [V exist]) to stay in sync with the library.

### 4.4 State design
```ts
ChartModel = Schema.Struct({
  size: Schema.Struct({ width: Schema.Number, height: Schema.Number }),   // from a ResizeObserver Mount (responsive)
  focus: Schema.Option(Schema.Struct({ seriesId: Schema.String, index: Schema.Int })),
  zoom: Schema.Option(Schema.Struct({ x0: AxisValue, x1: AxisValue })),   // brush/zoom domain
  hiddenSeries: Schema.Array(Schema.String),
})
// data is NOT copied into ChartModel; it's read from host Model/Remote via config selector
```
- **Messages:** `ChartResized`, `ChartFocused`, `ChartFocusCleared`, `ZoomChanged`, `ZoomReset`, `SeriesToggled`, and `PointActivated` → OutMessage.
- **Mirror:** `zoom` and `hiddenSeries` → URL (shareable dashboard state).
- **Scene memoization:** `createLazy` keyed on `(data ref, size, zoom, hiddenSeries, theme)`.

### 4.5 Theming
- **Client:** map pocket theme tokens → chart theme object (extend `defaultChartTheme`) and CSS variables for Option 2 so Mixins Styles apply.
- **Server:** resolve tokens to literal colors (email clients ignore CSS variables).

### 4.6 Server rendering service (pocket)
```ts
class Charts extends Context.Service<Charts, {
  renderSvg<D>(chart: ChartDefinitionFor<D>, data: ReadonlyArray<D>, opts: { width; height; theme? }): Effect.Effect<string, ChartRenderError>
  renderPng(...): Effect.Effect<Uint8Array, ChartRenderError>   // via resvg/sharp adapter for email clients lacking SVG support
}>()("pocket/Charts") {}
```
- **Integrations:** emailcn bento stats digest (sparklines + KPIs), pdfcn reports, OG images (stats cards).
- **Email caveat:** many clients (notably Gmail/Outlook) don't render inline SVG. Default to PNG (resvg adapter) with SVG for web views. Verify per client.
- **Performance:** pure function; cache via `PersistedCache` keyed by `(chart id, data hash, size, theme)`.

### 4.7 Accessibility
- `role="img"` + `<title>`/`<desc>` generated from spec; optional visually hidden data table (derived from the same data) and keyboard focus navigation across points (Messages `ChartFocusMoved`).

### 4.8 Milestones
1. Server `renderSvg` + Foldkit Option 1 static chart (line/bar/area) with theme tokens.
2. Option 2 scene-walker renderer, responsive size Mount, focus/tooltip via `findNearestPoint`.
3. Zoom/brush, legend toggles, URL mirror, keyboard navigation.
4. Email PNG path, PDF integration, pocket dashboard widgets (records over time, jobs, latency from OTel metrics).

---

## 5. `@pocket/foldkit-range`: ranger via Mount, value in Model

### 5.1 Goals
- **Slider kinds:** single and multi-handle range sliders (price range, date range, numeric filters in tables).
- **Steps and marks:** step size or explicit steps, ticks, min distance between handles.
- **Interaction and output:** keyboard accessible, pointer/touch drag; committed vs live (dragging) values.

### 5.2 Relevant ranger facts [V]
- **Constructor:** `new Ranger(config)`; `RangerOptions` = `RangerConfig` minus `rerender`, plus `stepSize` **or** `steps`.
- **Instance state:** `activeHandleIndex`, `tempValues` (during drag), `sortedValues`.
- **Methods:** `setOptions`, `_willUpdate`, `getValueForClientX`, `getNextStep`, `roundToStep`, `handleDrag`, `handleKeyDown(e, i)`, `handlePress(e, i)`, `getPercentageForValue`, `getTicks()`, `getSteps()`, `handles()` (value, isActive, onKeyDownHandler, onMouseDownHandler, onTouchStart…).
- **Config** (`RangerConfig` fields such as `getRangerElement`, `values`, `min`, `max`, `onChange`, `onDrag`, `interpolator`, `tickSize`) [U: confirm exact field names].
- Version **0.0.4**: API may change. Wrap behind our own interface; fallback to a small in-house implementation if needed.

### 5.3 State design
```ts
RangeModel = Schema.Struct({
  values: Schema.Array(Schema.Number),           // committed (source of truth, mirrorable)
  dragValues: Schema.Option(Schema.Array(Schema.Number)),   // live while dragging (not mirrored)
  activeHandle: Schema.Option(Schema.Int),
  focusedHandle: Schema.Option(Schema.Int),
})
RangeConfig = { min, max, stepSize | steps, minDistance?, ticks?: number | "steps", format: (n) => string, interpolator?: "linear" | "log", orientation: "horizontal" }
```

### 5.4 Mount design
```
Mount.defineStream("RangeTrack", { min, max, stepSize|steps, values: model.values })
  acquire:
    ranger = new Ranger({ getRangerElement: () => element, min, max, stepSize, values,
      onDrag:   (inst) => emit(RangeDragged({ values: inst.tempValues })),
      onChange: (inst) => emit(RangeCommitted({ values: inst.sortedValues })) })
    attach pointer/touch listeners on handle elements (found via data-handle-index) → ranger.handlePress(e, i)
  on args change: ranger.setOptions({ values: model.values, ... })  // Model stays owner
  release: remove listeners
```
- **Pure positions:** handle positions in `view` come from Model (`getPercentageForValue` is pure math; compute it in view with the same interpolator, or via a pure helper constructed without an element).
- **Keyboard without the Mount:** handled via `h.OnKeyDown` → `RangeKeyPressed({ handle, key })`; `update` computes the next value with the same step logic (`getNextStep`/`roundToStep` equivalents in a pure helper). Keyboard works in tests and SSR.

### 5.5 Messages
```
RangeDragStarted { handle }  RangeDragged { values }  RangeCommitted { values } → OutMessage ValueCommitted
RangeKeyPressed { handle, key }  HandleFocused { handle }  HandleBlurred
ValuesSet { values }   (external, e.g. from URL mirror or reset)
```
- `update` enforces invariants: sorted, clamped to `[min,max]`, snapped to steps, `minDistance` respected (property-tested with `Arbitrary`).

### 5.6 View & a11y
- **Slots:** `root`, `track`, `segment` (from `getSteps`), `fill`, `handle`, `tick`, `tickLabel`, `valueLabel`.
- **Handles:** `role="slider"`, `aria-valuemin/max/now`, `aria-valuetext` (format), `aria-orientation`, `tabindex=0`. Home/End/PageUp/PageDown/Arrow keys.
- **Table filter integration:** `ColumnFilterSet` receives a `[min, max]` value on `RangeCommitted` (not on every drag), so server queries only fire on commit.

### 5.7 Milestones
1. Pure step/clamp helpers + keyboard-only slider (no Mount) with tests.
2. Ranger Mount for drag; live vs committed values; replay safety.
3. Multi-handle, ticks, minDistance, log interpolator.
4. Table filter + URL mirror integration; date range variant (values as epoch ms with DateTime formatting).

---

## 6. `@pocket/foldkit-forms`: Foldkit validation + Effect Schema (form-core optional)

### 6.1 Goals
- **Data:** forms generated from pocket collection Schemas (create/edit records, settings) and hand-written forms.
- **Validation:** field-level and form-level, sync and async (unique email, rules dry-run), debounced async; decoded, typed output (Schema decode) on submit.
- **Structure:** arrays (repeatable fields), nested objects, relations, files, rich text.
- **Drafts and agents:** persisted drafts; agent-fillable.
- **form-core:** used only where it demonstrably saves work, and only in controlled mode.

### 6.2 Default design (no form-core)

**State**
```ts
FormModel<Encoded> = Schema.Struct({
  values: EncodedValues,                         // the Schema's *encoded* side (strings from inputs) → decoded on submit
  touched: Schema.Record(FieldPath, Schema.Boolean),
  dirty: Schema.Record(FieldPath, Schema.Boolean),
  errors: Schema.Record(FieldPath, Schema.Array(FieldError)),   // from SchemaIssue formatted per path
  asyncValidation: Schema.Record(FieldPath, AsyncStatus),        // Idle | Pending(token) | Valid | Invalid
  submit: SubmitStatus,                                          // Idle | Submitting | Succeeded | Failed(error)
  submitCount: Schema.Int,
})
```
- **Field paths** are typed from the Schema (`DeepKeys`-like helper over `Schema.Struct` fields, including array indices).
- **Values are the encoded representation,** so inputs bind naturally (`NumberFromString`, `DateFromString`, `Trimmed`). Decoding on validate/submit gives typed domain values: Schema codecs replace ad-hoc parsing.

**Validation pipeline**
1. `FieldChanged { path, value }` updates `values`, marks dirty, schedules validation per `validateOn` config (`change | blur | submit`).
2. Sync field validation uses **Schema per field** (derive a field schema from the struct; run `Schema.decodeUnknownExit`; map `SchemaIssue` → messages via `SchemaIssue` formatters + i18n).
3. Cross-field/form-level validation uses struct-level `check` filters (Schema filter groups) → errors attached to paths (`Pointer` issues [V issue types]).
4. Async validation: Command `Effect` (debounced with `Effect.sleep` + token cancellation; stale results ignored by token) calls pocket (`/api/collections/:c/validate` or rules dry-run) → `AsyncValidated { path, token, result }`.
5. `SubmitRequested`: full decode (`Schema.decodeUnknownExit(schema)(values)`). On success: Command performs mutation (foldkit-remote optimistic mutation / pocket Rpc) → `SubmitSucceeded | SubmitFailed`. Server validation errors (typed `TaggedError` from pocket with field paths) map back into `errors`.

Foldkit's built-in `fieldValidation` module [exists in foldkit exports; U: API] is used for the per-field status plumbing where it fits; the Schema layer above stays ours.

**Arrays & nesting**
- `ArrayItemAdded { path, value? }`, `ArrayItemRemoved { path, index }`, `ArrayItemMoved { path, from, to }` with path re-indexing of `touched/dirty/errors`.
- Stable item keys stored alongside values for rendering identity.

**Field components (Slots)**
- **Primitives:** `Field.text | textarea | number | select | multiSelect | checkbox | switch | date | dateRange (foldkit-range/calendar) | relation (combobox + foldkit-remote search) | file (pocket Storage upload with progress Commands) | richText (editor custom element) | json`
- **Slots:** `root`, `label`, `control`, `description`, `error`, `requiredMark`.
- **Error display policy:** show after touched or submitCount > 0 (configurable).

**Generated forms from pocket collections**
- `Pocket.formSpec(collectionSpec, { mode: "create" | "update" })`: uses the collection's insert/update Schema variants (`Model.Class` variants [V exist in unstable/schema]), annotations (`title`, `description`, `examples`) for labels/help, field type → field component, rules-derived read-only fields.
- **Server-side reuse:** the same Schema validates on the server, so client and server errors are identical in shape.

**foldkit-plus**
- **Mirror.kv:** drafts (`values` only, per form id/record id). Restore prompt on load; cleared on success.
- **Surface/Agent:** projection = field values + errors + schema description (from annotations); Messages = `FieldChanged`, `ArrayItemAdded/Removed`, `SubmitRequested`. Agents fill forms through the same validation and submit path (rules enforced server-side).
- **Mixins:** layout variants (stacked, inline, grid) and design-system styles without touching field logic.

### 6.3 When to use form-core (optional adapter)
Use `@tanstack/form-core` 1.33.5 only if the default design proves costly for:
- Complex **linked fields** (field A's validation re-runs when B changes) at scale.
- Large dynamic **arrays** with per-item async validation and debouncing (form-core bundles `pacer-lite` [V dep]).
- Interop with existing TanStack Form schemas in a migrating codebase.

**Controlled-mode adapter** (`@pocket/foldkit-forms/tanstack`):
- **Validation-engine role:** form-core acts as a *validation/metadata engine*. Foldkit Model remains the owner of `values`, `touched`, `errors`.
- **Instance lifetime:** create the `FormApi` instance inside a Mount/Subscription scope. Feed values from Model on each change (`form.setFieldValue` with "don't notify" semantics [U]). Subscribe to its store (`@tanstack/store` [V dep]); translate meta/errors changes into `FormEngineUpdated { errors, meta }` Messages.
- **Validators:** Effect Schema via Standard Schema (`Schema.toStandardSchemaV1` [V]). Async validators as Promises that run Effects through a `ManagedRuntime`, cancelled on scope close.
- **Replay/SSR:** without the Mount, the default Schema pipeline still validates, so form-core is an **enhancement**, never required for correctness.
- **Guardrail:** if form-core needs to own `values` to function, don't use it (anti-pattern: second owner).

### 6.4 Milestones
1. Default engine: flat struct form, encoded values, sync Schema validation, submit decode, Scene tests.
2. Async validation with debounce/cancellation; server error mapping; foldkit-remote mutation.
3. Arrays/nesting, relation/file/date-range fields, Slots + Styles.
4. Collection-generated forms, KV drafts, Surface/Agent.
5. (Conditional) form-core controlled adapter spike; adopt only if benchmarks/complexity justify.

---

## 7. Cross-cutting roadmap (joint with foldkit-bundle; supersedes the previous ordering)

Bundle phases (B0–B5, 5b) refer to [foldkit-bundle-implementation-plan.md](foldkit-bundle-implementation-plan.md). Component milestones (M1…) refer to sections 1–6 here.

| Order | Deliverable | Depends on | Notes |
|---|---|---|---|
| 1 | **Bundle B0** verification spike + **components §8 spikes** in parallel (Mount args semantics, table-core controlled state, virtual-core lifecycle, charts scene, ranger config) | — | One combined spike week; answers most [U] items for both plans |
| 2 | **Bundle B1** core single placement (`make`, `Link`, `at`, `fromParts`) | 1 | Parity baselines: MediaQuery, Pagination, Listbox |
| 3 | `shared` components foundation: Mount utils, rAF batching, Parts → Slots derivation, engine conformance harness, **vanilla-bundle vs plus-placement parity harness**, theme tokens | 2 | Parity harness reuses bundle parity tests |
| 4 | **`range` M1–M2 as the first real component bundle** (in-house vs ranger engine; keyboard pure; drag Mount) | 2, 3 | Small, nests into Table/Forms, so it exercises `Link.compose` early |
| 5 | **Bundle B2** assembly & completeness (+ C4 ergonomics: auto wrapper, `modelFields`, `Bundle.place`) | 2 | Range placement is the test fixture |
| 6 | `hotkeys` M1–M2 as a bundle (global registrations as bundle subscriptions; scoped Behavior/Mount) | 5 | Proves subscription lifting + gating in a real component |
| 7 | `virtual` M1–M2 as a bundle | 5 | |
| 8 | **Bundle B3** collections (`each` v1; v2 if spike go) | 5 | Needed for Table filters, chart grids, form groups |
| 9 | `table` M1–M3 as a bundle: nests Virtual (`Link.compose`), Range filters (`each`), scoped hotkeys; server mode (vanilla Command path) | 4, 6, 7, 8, pocket Records API | **Headline example** for bundle RFC Experiment G/H evidence |
| 10 | **Bundle B4** bundle-surface: contracts, relative Surfaces, Mirrors | 5 | Unlocks plus path for all components |
| 11 | Components `/plus` layers: Table/Range/Forms Surfaces + Mirrors, Table server mode via foldkit-remote | 9, 10 | Two tables on one page = two tool sets + two URL namespaces |
| 12 | `forms` M1–M3 as a bundle (Listbox via `fromParts`, field groups via `each`) | 8, 10, pocket validation endpoints | |
| 13 | `charts` M1–M2 as a bundle (+ server `Charts` service, not a bundle) | 5, pocket metrics API | Dashboard grid via `each` |
| 14 | **Bundle 5b** primitives seed (MediaQuery, Online, Timer, Pagination, History, WebSocket) in `@pocket/foldkit-primitives` | 5 | Parallel track |
| 15 | Pocket admin assembly: records browser + dashboard + shortcuts as one `Bundle.assemble`; `Module.validate` + `toMermaid` in CI | 9–13 | |
| 16 | **Bundle B5** examples/docs/release + **RFC Part D** PR (using Table/Range/Forms evidence) | 9, 11, 15 | |
| 17 | Advanced milestones (table M4–M5, charts M3–M4, forms M4–M5, hotkeys M3–M4) + registry items + docs | — | |

**Critical path:** B0 → B1 → Range → B2 → (Hotkeys, Virtual, B3) → Table → B4 → plus layers → Forms/Charts → admin assembly → release + RFC.

## 8. Open questions / verification list
0. Architecture: confirm the core/plus split (0.5a) and engine abstraction (0.5b) don't force duplicate view code; decide whether engines are plain config objects only or also Effect Layers for every package (currently: config for client, Layer where server use exists).
1. Foldkit: Mount args-update semantics (re-acquire vs update), Subscription keying on args, raw SVG/HTML insertion API, `fieldValidation` API, custom Mixins Capabilities, Behavior attach API, Mirror API for nested Submodel fields.
2. table-core v9: controlled `state` + change handler shape vs `atoms` (`ExternalAtoms`) contract; manual (server) mode option names; pure "next state" helpers per feature; `static-functions` entry purpose.
3. virtual-core 3.17: mount/update lifecycle method names (`_didMount`, `_willUpdate`, `setOptions`), scroll APIs (`scrollToIndex`, `scrollToOffset`), measurement flow.
4. hotkeys 0.8: scoped target option vs local handler; sequence manager registration API.
5. charts 0.18: scene node types stability for a Foldkit walker; `RenderChartSvgOptions`; server runtime without DOM (d3-selection import side effects); email PNG path.
6. ranger 0.0.4: exact `RangerConfig` fields; stability. Decide wrap vs in-house.
7. form-core 1.33: controlled usage feasibility (values ownership).
8. foldkit-plus: remote query/mutation API for server-mode table and forms; Surface helper composition for Submodels.
9. Bundles: can Mounts inside a bundle view be declared so `/plus` derives Behaviors automatically? Does `Link.compose` preserve per-placement namespacing for nested Mirrors (`?records.filters.amount=…`)? Are hotkeys registry Messages better routed as bundle OutMessages or directly to host Messages?
