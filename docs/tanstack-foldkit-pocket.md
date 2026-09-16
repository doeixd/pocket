# TanStack × Foldkit / foldkit-plus × pocket

Drafted 2026-09-16. Inputs: [tanstack-ecosystem.md](tanstack-ecosystem.md), [react-in-foldkit.md](react-in-foldkit.md), [foldkit-to-react.md](foldkit-to-react.md), [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md), foldkit-plus README (doeixd/foldkit-plus). TanStack/foldkit-plus API details beyond the READMEs are **unverified**; check before implementing.

## 1. The governing rule: one owner per datum
foldkit-plus's core rule: **every piece of state has exactly one owner.**
| State | Owner | Package |
|---|---|---|
| Route, selection, transient error | local Model (`update`) | Foldkit |
| URL filter/page, device draft/prefs | local Model | `foldkit-mirror` observes |
| Server-owned facts | server | `foldkit-remote` caches |
| Client-authored state that must converge offline | durable log | `foldkit-sync` + `foldkit-durable` |
| What an agent may see/do | application | `foldkit-agent` |

Most TanStack libraries **own state internally** (Query cache, DB collections, Form store, Table state via `@tanstack/store`). Dropped naively into Foldkit, they create a second owner. So the integration rule is:

> **Use TanStack cores as pure computation or controlled engines. State lives in the Foldkit Model (or a foldkit-plus owner); TanStack computes derived results and emits events that become Messages.**

## 2. foldkit-plus already covers several TanStack roles
| TanStack | foldkit-plus equivalent | Decision in Foldkit apps |
|---|---|---|
| **Query** (server state cache, invalidation, optimistic mutations) | **`foldkit-remote`**: normalized server entity cache, "know what is missing", optimistic mutation, live updates; `foldkit-remote-server` + `foldkit-remote-drizzle` on the server | **Use foldkit-remote.** Don't use Query in Foldkit |
| **DB** (collections, live queries, optimistic, offline sync backends) | **`foldkit-remote`** (server-owned) + **`foldkit-sync`/`foldkit-durable`** (offline, multi-device, converge by replaying Messages) | **Use foldkit-plus.** Borrow ideas (live-query IVM, persistence adapters) |
| **Router** search-param state | **`foldkit-mirror`** `Mirror.url` (Model fields mirrored to URL) + Foldkit `route`/`navigation` | **Use Foldkit + Mirror** |
| **Persist** / query persisters | **`foldkit-mirror`** `Mirror.kv` (KeyValueStore) | **Use Mirror** |
| **AI** client hooks (chat/tools UI) | **`foldkit-agent`** (+ `-mcp`, `-webmcp`, `-a2a`, `-native`): app Messages as agent tools with authorization | **Use foldkit-agent** for app-as-tool; see §4.6 for TanStack AI server-side |
| **Workflow** | server-side `effect/unstable/workflow` (pocket) | Not a client concern |
| **Store** | Foldkit Model | Not needed |
| **DevTools** | Foldkit DevTools (time travel, Mount history) | Foldkit's; optional TanStack DevTools plugin for React hosts |

## 3. TanStack cores that fill real gaps in Foldkit
| Need | TanStack core | Integration pattern | foldkit-plus touchpoints |
|---|---|---|---|
| **Data grid** (sort, filter, group, paginate, pin, resize, select, expand) | `@tanstack/table-core` 9.x | **Controlled pure engine** (§4.1) | Mirror.url for sort/filter/page; Mixins Slots for cell/row/header styling; Surface to expose table actions to agents; Remote supplies rows |
| **Virtualized lists/grids** | `@tanstack/virtual-core` 3.x | **Mount** measures scroll/size → Messages with visible range (§4.2) | Mixins Behavior on the scroll container Slot |
| **Complex forms** (arrays, linked fields, async validation) | `@tanstack/form-core` 1.x | Prefer Foldkit `fieldValidation` + Schema; use form-core only as a controlled validator engine if needed (§4.3) | Mirror.kv for drafts; Agent can fill forms via existing Messages |
| **Keyboard shortcuts** | `@tanstack/hotkeys` (core) | **Subscription** (document-level) or Mount (scoped) → Messages (§4.4) | Surface declares which Messages are hotkey-reachable; Mixins Behavior for scoped hotkeys |
| **Range sliders** | `@tanstack/ranger` 0.0.x | Mount for pointer handling, value in Model | Mixins Slots for track/handle styling |
| **Charts** | `@tanstack/charts` 0.18 | Pure: data from Model → SVG output → Foldkit view (or server-render) (§4.5) | Mixins Styles for theme tokens |
| **Debounce/throttle/queue in UI** | `@tanstack/pacer` | Prefer Effect (`Stream.debounce`, `Schedule`) in Commands/Subscriptions; Pacer only if already used | — |
| **Markdown / highlight** | `@tanstack/markdown`, `@tanstack/highlight` | Pure: string → HTML/tokens; render via Foldkit (raw HTML insertion API: verify) | — |
| **Select / Time (calendar)** | not yet published | Watch; meanwhile Foldkit `calendar`, Zag machines (react-in-foldkit §3) | — |

## 4. Integration patterns (sketches; verify APIs)

### 4.1 Table: controlled engine
```ts
// Model owns table state; table-core computes the row model.
const Model = Schema.Struct({
  sorting: SortingState, columnFilters: FiltersState, pagination: PaginationState,
  rowSelection: Schema.Record(Schema.String, Schema.Boolean),
  // rows themselves come from foldkit-remote (server-owned)
})

const view = (model, h) => {
  const table = createTable({                 // table-core; verify v9 constructor (features are opt-in in v9)
    data: Remote.select(model, RecordsQuery),  // server-owned rows
    columns,
    state: pick(model, ["sorting", "columnFilters", "pagination", "rowSelection"]),
    onStateChange: () => {},                   // never mutate here; events dispatch Messages below
    getCoreRowModel: getCoreRowModel(), getSortedRowModel: getSortedRowModel(),
  })
  return TableView(table, h)                   // header click → h.OnClick(Message.SortedBy({ id }))
}
// update applies SortedBy etc. to Model; Mirror.url(App, { fields: [sorting, pagination] }) makes it linkable
```
- Build the table instance per render (or memoize with `createLazy` keyed on inputs); cheap for typical sizes.
- **Server-side mode** (pocket): `manualSorting/manualFiltering/manualPagination`; the Model's table state is the query sent through **foldkit-remote** to pocket's Records API (filter language). The same state object drives both.
- **Mixins:** publish Slots `table.root/header/row/cell/pagination`; pocket admin themes via Styles; users restyle without forking.
- **Agent:** Surface `RecordsTable` exposes `SortedBy`, `FilteredBy`, `SelectedRows`, `DeletedSelected` Messages as tools, so "delete all unpaid invoices older than 30 days" runs through the same update and rules.

### 4.2 Virtual: Mount + visible range in Model
- `Mount.defineStream("VirtualScroller", …)`: acquire a `Virtualizer` from virtual-core on the scroll element, observe scroll/resize, emit `Message.VisibleRangeChanged({ start, end, totalSize })`, release on unmount.
- `update` stores the range; `view` renders only items in range with top/bottom spacers (`totalSize`).
- Dynamic heights: a per-item Mount reports measured size → `Message.ItemMeasured`.
- **pocket admin:** logs viewer, realtime record feeds, trace spans list.

### 4.3 Forms: prefer Foldkit, optionally form-core
- **Default:** Foldkit Model holds field values; `fieldValidation` + Effect Schema (collection Schema → field validators); async validation as Commands calling pocket (unique checks, rules dry-run).
- **Optional form-core:** use as a validation/dirty-tracking engine in controlled fashion (values from Model, validators via Standard Schema from `Schema.toStandardSchemaV1`). Only if its array/linked-field logic saves real work; otherwise a second owner.
- Mirror.kv persists drafts; foldkit-agent can fill forms by sending the same field-changed Messages.

### 4.4 Hotkeys: Subscription → Messages
- Global: a Foldkit `Subscription` wrapping hotkeys core registration, emitting `Message.HotkeyPressed({ id })`; `update` maps ids to actions.
- Scoped: Mixins **Behavior** attaches a Mount on a Slot (e.g. `table.root`) registering scoped hotkeys; unregistered on unmount via finalizer.
- Declare a typed `Hotkeys` map per Surface so the admin can render a shortcuts help panel and agents can discover the same actions.

### 4.5 Charts: pure SVG
- Chart spec + data from Model → charts core produces scales/marks → render with Foldkit's SVG builder (verify whether @tanstack/charts outputs SVG strings, a scene graph, or requires a DOM).
- Server-render the same chart for **emails/PDFs** (pocket `EmailTemplates`/`Documents`), e.g. weekly stats digest via the emailcn bento stats grid.

### 4.6 TanStack AI (server-side, pocket)
- Not a Foldkit client concern. In pocket, `LanguageModel` adapter over `@tanstack/ai` providers; tools come from pocket collections and from **foldkit-agent** contracts (the admin app's Messages served over MCP).

## 5. pocket mapping: which layer owns what
| pocket capability | Server (Effect) | Foldkit admin / Foldkit user apps | React user apps |
|---|---|---|---|
| Records read/cache | Records HttpApi/Rpc | **foldkit-remote** (+ pocket implements the `foldkit-remote-server` protocol; `foldkit-remote-drizzle` matches pocket's Drizzle layer) | **`@pocket/tanstack-db`** collection or TanStack Query via effect-query |
| Realtime | Realtime service (Rpc stream / SSE) | foldkit-remote live updates fed by pocket stream | TanStack DB collection sync |
| Offline / multi-device | `foldkit-durable` journal hosted by pocket (per collection/doc), rules as `authorize` | **foldkit-sync** | TanStack DB `offline-transactions` + SQLite persistence (pocket collection) |
| Lists/grids | Filter language + pagination | table-core (controlled) + virtual-core + Mirror.url | `@tanstack/react-table` + react-virtual (registry block) |
| Forms | Collection Schema, rules dry-run endpoint | Foldkit fieldValidation (+ optional form-core) + Mirror.kv drafts | `@tanstack/react-form` with Standard Schema (registry `record-form`) |
| Routing | — | Foldkit route + Mirror.url | TanStack Router / Start |
| Hotkeys | — | hotkeys core via Subscription/Behavior | `@tanstack/react-hotkeys` |
| Charts / dashboards | Metrics & aggregates API | charts core → SVG | `@tanstack/charts` |
| AI / agents | `unstable/ai` (+ TanStack AI adapter), McpServer | **foldkit-agent** (+ mcp/webmcp adapters) exposes admin Messages | TanStack AI React hooks against pocket endpoints |
| Theming/customization | tokens in `pocket.config.ts` | **foldkit-mixins** (+ `-ui` for `@foldkit/ui`) | shadcn CSS variables |
| Architecture docs | — | `Module.validate` / `Module.toMermaid` over admin Surfaces | — |
| Host framework | pocket server | Foldkit SPA served by pocket | **TanStack Start** template mounting pocket via `toWebHandler` |

### Key opportunities
1. **pocket speaks foldkit-remote and foldkit-durable natively.** pocket's Records service implements `foldkit-remote-server`; pocket hosts `foldkit-durable` journals with collection rules as `authorize`. Foldkit apps get cache, optimistic updates, live updates and offline sync with zero glue. Verify protocol shapes in `docs/remote.md` / `docs/replication.md`.
2. **One contract, two client stacks.** The same Records/Realtime protocol backs foldkit-remote (Foldkit) and a TanStack DB collection (React/Solid/Vue). Parity tests run both clients against pocket.
3. **`@pocket/foldkit-table`:** table-core + virtual-core + Mirror.url + Mixins Slots + Surface/Agent actions, driven by collection metadata. This is the admin records browser, and it ships in the registry.
4. **Agent-operable admin:** every admin Surface (records table, forms, settings) is exposed via foldkit-agent to MCP, using the same Messages, rules and audit trail.

## 6. Anti-patterns
- Using TanStack Query or DB **inside** a Foldkit app alongside foldkit-remote/sync (two caches, two owners).
- Letting table-core/form-core keep **uncontrolled** internal state.
- Mutating Model from TanStack callbacks instead of dispatching Messages.
- Putting URL state in TanStack Router *and* Mirror.url.

## 7. Next steps
- Read foldkit-plus `docs/remote.md` and `docs/replication.md`; draft the pocket ↔ foldkit-remote-server / foldkit-durable protocol mapping.
- Spike `@pocket/foldkit-table`: table-core v9 controlled + virtual-core Mount + Mirror.url + Mixins Slots on a pocket collection.
- Spike `@pocket/tanstack-db` collection against the same Records/Realtime API.
- Verify: table-core v9 instance creation, virtual-core Virtualizer options, hotkeys core registration API, charts output format, Foldkit raw-HTML API.
