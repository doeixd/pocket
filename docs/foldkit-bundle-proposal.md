# Proposal: Bundles and Links for Foldkit (a foldkit-plus RFC draft)

Drafted 2026-09-16. Status: design exploration, intended as an RFC for doeixd/foldkit-plus. Code sketches are **not compiled**; signatures they build on were read from `foldkit@0.160.0` and `@foldkit/ui@0.160.0` type declarations and foldkit-plus source/docs (`docs/rfc.md`, `docs/design/REVISION_PLAN.md`, `packages/surface`, `packages/sync`, `packages/remote`).

Related: [foldkit-primitives.md](foldkit-primitives.md), [foldkit-tanstack-components-design.md](foldkit-tanstack-components-design.md).

---

## 1. The observation

Foldkit already has **four separate lifts**, and they all take the same information:

| Where | Foldkit API (verified) | What it needs |
|---|---|---|
| update | `Update.foldChild({ update, read, write, toParentMessage, foldOutMessage? })` | `read: Parent → Option<Child>`, `write: (Parent, Child) → Parent`, `toParentMessage` |
| view | `h.submodel({ slotId, model, view, toParentMessage, viewInputs? })` (`SubmodelConfig`) | child model (read), `toParentMessage` |
| subscriptions | `Subscription.lift(subs)({ toChildModel, toParentMessage, when? })` | read, `toParentMessage`, gate |
| resources | `ManagedResource.lift(resources)({ toChildModel: Parent → Option<Child>, toParentMessage })` | read (Option), `toParentMessage` |

So **a lens (read/write, Option-aware) + a Message wrapper + an optional gate** is the single idea behind all child composition in Foldkit, but it's written by hand 3–4 times per child, per parent.

Meanwhile the *child side* is also scattered:
- **`@foldkit/ui` already has a partial bundle.** `Listbox.create<Item>()` returns a `Bundle` = `{ view, update, selectItem, open, close }` "behind a single Item-typed entry point… so the view's `Item` type and the update's OutMessage `item` type can't drift." It covers view + update + helpers; subscriptions/resources and placement are left to the parent.
- **foldkit-plus rebuilds the same thing ad hoc per package.** `Sync.mount(app, sync, options)` wraps update + `Subscription.make/aggregate`. `Remote.subscriptions(activeSurfaces)` returns entries "keyed for `Subscription.make`" for the parent to aggregate.

**Failure mode today:** forget to `aggregate` a child's lifted subscriptions or resources and nothing complains; the feature silently doesn't work. `Module.validate` checks ownership of Model paths, not wiring.

## 2. Constraints (from foldkit-plus's own RFC)
The design must pass the bar foldkit-plus set for itself (`docs/rfc.md` §1):
1. **"Add a primitive only when it makes several existing or future features simpler at once."**
2. **No hidden state:** no new reducer, store, render loop, ownership system or async library.
3. **"Higher-level features should compile into existing effect categories"** (Command, Subscription, Mount, ManagedResource, Flag).
4. **Optional:** plain `Model/Message/init/update/view` apps stay complete.
5. **Writes still mean Messages.**
6. **Surface = access boundary; Submodel = ownership boundary.** Keep them distinct.

The proposal below adds **two values and zero runtime**. Everything compiles to `Update.foldChild`, `h.submodel`, `Subscription.lift/aggregate`, `ManagedResource.lift/aggregate`.

## 3. The idea: Bundle + Link = Placed
```
Bundle   what an ownership unit IS      (Model, Message, init, update, subscriptions, resources, view, helpers)
Link     where it LIVES in a parent     (lens read/write + toParentMessage + optional gate)
Placed   Bundle.at(Link)                (the same parts, lifted to the parent: native Foldkit values)
```

### 3.1 Bundle
A plain, inspectable descriptor. Everything optional except Model/Message/init/update.
```ts
export const MediaQuery = Bundle.make({
  name: "MediaQuery",
  Model: Schema.Struct({ matches: Schema.Boolean }),
  Message: MediaQueryMessage,                   // tagged union
  init: (args: { query: string }) => ({ matches: false }),
  update: (model, message) => MediaQueryMessage.match(message, { Changed: ({ matches }) => ({ model: { matches } }) }),
  subscriptions: (args) => Subscription.make<Model, Message>()((entry) => ({
    changes: Subscription.persistent(matchMediaStream(args.query)),
  })),
  // resources?, view?, OutMessage?, helpers?, surfaces?
})
```
- `Bundle.make` is the generalization of `@foldkit/ui`'s `create()`: it **locks the types together once** (Model, Message, OutMessage, requirements `R`, view inputs).
- **Args:** bundles are parameterized by `args` (a plain value; optional Schema for serializability/inspection). Args flow into `init`, `subscriptions`, `resources`, `view`.
- **Adapter:** `Bundle.fromParts({ Model, Message, init, update, view })` wraps existing modules and `@foldkit/ui` bundles without rewriting them.

### 3.2 Link
One value capturing "where and how":
```ts
// vanilla Foldkit
const darkLink = Link.make({
  read: (m: AppModel) => Option.some(m.prefersDark),
  write: (m, child) => evo(m, { prefersDark: () => child }),
  toParentMessage: (message) => GotPrefersDarkMessage({ message }),
  when: (m) => true,                       // optional gate for subscriptions/resources
})

// foldkit-plus: from a ModelRef/FieldRef (Optic-backed get/set already exist in foldkit-surface)
const darkLink = Link.field(App.fields.prefersDark, GotPrefersDarkMessage)
```
- `read` returns `Option` (children may be absent: routes, optional panels), matching `foldChild` and `ManagedResource.lift`.
- **Message wrapper helper:** `Link.wrapper("GotPrefersDarkMessage", MediaQuery)` builds the parent Message variant (Schema) *and* the `toParentMessage` function, so the name and codec can't drift.

### 3.3 Placed
```ts
const Dark = MediaQuery.at(darkLink, { query: "(prefers-color-scheme: dark)" })

Dark.Message        // the parent-side wrapper variant (Schema) to include in the parent's Message union
Dark.init           // (parentModel) → parentModel with child initialized (or use in parent init)
Dark.update         // Update.Fold (from Update.foldChild), OutMessage handled via options below
Dark.subscriptions  // lifted record (Subscription.lift)
Dark.resources      // lifted record (ManagedResource.lift)
Dark.view           // (parentModel, h, viewInputs) → Html via h.submodel
Dark.helpers        // child helpers lifted to parent steps (e.g. Listbox.open → Step<Parent>)
```
All fields are **native Foldkit values**. Nothing new at runtime.

### 3.4 Assembly: making wiring complete by construction
```ts
const Placements = Bundle.assemble([Dark, Uploads, Search])   // pure

// parent update: route Placed messages in one call
update: (model, message) =>
  Placements.route(model, message)            // Option<Return>: Some if the message belongs to a placed bundle
    .pipe(Option.getOrElse(() => ownUpdate(model, message)))

subscriptions: Subscription.aggregate<AppModel, AppMessage>()(ownSubs, Placements.subscriptions)
resources:     ManagedResource.aggregate<AppModel, AppMessage>()(ownResources, Placements.resources)
```
- `assemble` merges all placed subscriptions/resources (keys namespaced by bundle name + link path, so `aggregate`'s duplicate-key check still protects).
- **Type-level completeness (optional, stronger):** `Runtime.makeApplication` is wrapped by `Bundle.application({ …, placements })`, which *requires* `subscriptions`/`resources` to include `Placements`' (branded record types) and requires the parent Message union to include every `Placed.Message`. Forgetting one becomes a **compile error**.
- **OutMessages:** `Bundle.at(link, args, { onOut: (out, ctx) => Step<Parent> })` maps to `foldChild`'s `foldOutMessage`; `ctx` gives `liftCommand` (existing `FoldContext`).

## 4. Collections: `Bundle.each`
The case Foldkit makes hardest today (d.ts examples hand-roll `GotEntryMessage({ entryId, message })`).
```ts
const Uploads = Upload.each(
  Link.collection({
    read: (m) => m.uploads,                          // HashMap<UploadId, Upload.Model> | Record
    write: (m, uploads) => evo(m, { uploads: () => uploads }),
    toParentMessage: (key, message) => GotUploadMessage({ key, message }),
  }),
  { args: (key, child) => ({ endpoint: "/api/files" }) },
)
Uploads.update       // routes by key; missing key → no-op (typed Ignored)
Uploads.view         // (model, h, key) → Html   and  .viewAll(model, h, render)
Uploads.subscriptions
Uploads.resources
Uploads.add / remove // Steps that insert/remove a child (init + cleanup semantics)
```
**Lifecycle compilation (the hard part):**
- **Subscriptions** compile to **one** entry per bundle subscription key:
  - `modelToDependencies` = `HashMap<Key, ChildDeps>` (only keys whose gate passes)
  - `dependenciesToStream` = merge of per-key child streams
  - `keepAliveEquivalence` (verified to exist) keeps unchanged keys' streams alive across Model changes, so adding one upload doesn't restart the others. **Verify** whether `keepAliveEquivalence` + `readDependencies` (verified signature) is sufficient for per-key diffing, or whether a small `Stream` diff operator (`Stream.mergeByKey`) is needed.
- **ManagedResources** are **static keyed entries** in Foldkit today (`Entry` per key, requirements per entry). Per-item resources (a socket per chat room) likely need either:
  - (a) a resource whose value is itself a keyed pool (`RcMap`-backed service acquired once; per-key acquire/release via Commands/Subscriptions), or
  - (b) an upstream Foldkit addition: `ManagedResource.each`.

  Proposal: ship (a) in foldkit-plus; propose (b) upstream with evidence.

## 5. How it composes with foldkit-plus (plus-native, still optional)
| foldkit-plus concept | Bundle integration |
|---|---|
| **ModelRef / FieldRef** | `Link.field(ref, wrapper)`: lens comes from the Optic already stored in the ref |
| **Module / ownership** | A Placed bundle is a **contract of kind `"bundle"` owning its link path**. `Module.make(App, [Dark, Uploads, …])` → `Module.validate` now catches two bundles placed on the same path, or a bundle path also owned by Sync/Remote, *for free*. `Module.toMermaid` shows bundle placement |
| **Wiring validation** | New finding kind: "bundle placed but its subscriptions/resources not aggregated" (runtime check in `Bundle.application`, and in `Module.validate` when given the app's records) |
| **Surface** | A Bundle may declare **relative Surfaces** (projection over its own Model + its own Messages). `Placed.surfaces` re-roots them through the Link's optic and wraps Messages with `toParentMessage`. Every placed instance gets agent/mirror-ready Surfaces without redeclaring (e.g. each table on a page exposes its own `SortingSet` tool) |
| **Mirror** | Bundles declare **mirrorable fields** (relative refs + codecs); `Placed.mirrors.url(prefix)` / `.kv(key)` re-root them. URL param namespacing by placement (`?invoices.sort=…`) |
| **Mixins / SurfaceView** | Bundle `view` may be a `SurfaceView`; Placed view keeps Slots, so Styles/Behaviors attach per placement |
| **Agent** | `Agent.forApplication(App)` can include `Placed.surfaces`; tool names namespaced by placement |
| **Sync / Remote** | Their existing `mount`/`subscriptions` builders could *return Bundles* (e.g. `Remote.bundle(…)`), making them placeable like anything else, which also serves the "cross-package API consistency" goal in `docs/improvements.md` |

## 6. Why this is the "elegant, small" thing
- **Two nouns** (Bundle, Link) and one verb (`at`/`each`), no new effect category, no runtime.
- **It removes repetition that already exists** in core (four lifts), `@foldkit/ui` (Bundle), and foldkit-plus (`Sync.mount`, `Remote.subscriptions`), which is the RFC's bar: "makes several features simpler at once."
- **Ownership stays explicit:** a Bundle *is* a Submodel (ownership boundary), just packaged. Surfaces stay the access boundary.
- **Type-driven correctness:** Model/Message/OutMessage/R/args locked at definition; wiring completeness checkable at the application boundary.
- **Enables the ecosystem:** primitives (media query, websocket, pagination, tween), TanStack-backed components (table, virtual, range), `@foldkit/ui` components, and pocket features all become "define once, place anywhere, place many."

## 7. Naming
| Candidate | Pros | Cons |
|---|---|---|
| **Bundle** (recommended) | **Already Foldkit vocabulary** (`@foldkit/ui` `Bundle`, `create()`); literally "parts bundled behind one typed entry point" | Slightly generic |
| Submodel.define / Submodel.at | No new noun; honest (it *is* a Submodel) | "Submodel" today names a pattern, not a value; overloading may confuse |
| Unit / Piece / Capsule / Organ | Evocative | New vocabulary with no precedent |
| Component | Familiar | Implies view-centric; many bundles have no view (media query, websocket); clashes with UI-library meaning |
| Part | Short | Taken by Mixins/Parts terminology |
| Feature | Business-friendly | Surface docs already use "feature" loosely for boundaries |

For the placement side: **Link** (or `At`/`Placement`). Result type: **Placed** (or `Mounted`, but *Mount* is taken).

## 8. Examples

**Stream-only primitive** (no view)
```ts
const Online = Bundle.make({ name: "Online", Model, Message, init: () => ({ online: true }), update,
  subscriptions: () => Subscription.make()(() => ({ status: Subscription.persistent(onlineStream) })) })
const OnlineHere = Online.at(Link.field(App.fields.online, GotOnlineMessage))
```

**Resource primitive** (websocket; gated by Model)
```ts
const Socket = Bundle.make({ name: "Socket", …,
  resources: (args) => ManagedResource.make()((entry) => ({
    socket: entry({ modelToMaybeRequirements: (m) => m.wanted ? Option.some({ url: args.url }) : Option.none(), … }) })) })
const Chat = Socket.at(Link.field(App.fields.chatSocket, GotChatSocketMessage), { url: "wss://…" })
```

**Existing @foldkit/ui bundle**
```ts
const ColorListbox = Bundle.fromParts(Listbox.create<Color>(), { Model: Listbox.Model, Message: Listbox.Message, init: Listbox.init })
const Picker = ColorListbox.at(Link.field(App.fields.colorPicker, GotColorPickerMessage), {}, {
  onOut: (out) => Listbox.OutMessage.match(out, { Selected: ({ item }) => setColor(item) }) })
```

**TanStack-backed table** (pocket)
```ts
const InvoicesTable = Table.bundle<Invoice>({ columns, engine: TanStackTableEngine, mode: "server" })
  .at(Link.field(App.fields.invoicesTable, GotInvoicesTableMessage))
// Placed.surfaces → agent tools; Placed.mirrors.url("invoices") → linkable state
```

## 9. Vanilla vs plus
- **`foldkit-bundle` (core-only):** `Bundle.make/fromParts/at/each/assemble/application`, `Link.make/collection/wrapper`. Depends only on `foldkit` + `effect`.
- **plus integration** (in `foldkit-surface`, or a `foldkit-bundle/plus` subpath): `Link.field`, Module contract kind, relative Surfaces/Mirrors, Agent namespacing.
- Matches the dual-support rule in the pocket components plan: plus derives, never adds core behavior.

## 10. Non-goals
- No hidden child state, no per-bundle runtime, no dependency injection container.
- Not a replacement for `h.submodel`/`foldChild`: those remain the primitives; Bundles call them.
- No automatic discovery (placements are explicit values, consistent with "no hidden global registry").

## 11. Open questions
1. Can `keepAliveEquivalence` + `readDependencies` express per-key stream lifetimes for `each`, or do we need a keyed-merge Stream operator?
2. Per-item ManagedResources: pool-as-resource (RcMap) in plus vs `ManagedResource.each` upstream.
3. `h.submodel`'s `slotId` for collections (key-derived ids) and DevTools labeling.
4. Type performance: 15-parameter Schema types + lifted records across many placements. Measure `tsc` on a 30-placement app.
5. `init` composition: explicit `Placed.init(parentModel)` steps vs deriving the parent Model Schema from placements (`Bundle.modelFields(placements)`) to avoid redeclaring child fields.
6. Declaration emit portability (foldkit-plus `improvements.md` notes TS2742 issues): publish nameable `Bundle`/`Placed` types from day one.
7. Should Flags and Ports participate (a bundle declaring required flags/ports)?
8. Where the upstream line sits: Link + `Bundle.at` could plausibly live in Foldkit core (it only packages core lifts); `each`, Module/Surface/Mirror integration in foldkit-plus.

## 12. Prototype plan
1. **Spike (1–2 days):** `Bundle.make`, `Link.make`, `.at` for update/view/subscriptions/resources; port `MediaQuery`, `Pagination`, and wrap `Listbox.create` with `fromParts`. Assert zero behavior change vs hand-wired versions (same Message log).
2. **Assembly:** `assemble` + `Bundle.application` compile-time completeness check; negative type tests (`.test-d.ts`) proving a missing aggregate fails.
3. **Collections:** `each` with subscriptions keep-alive per key; measure restarts; decide on the resource-pool approach.
4. **plus:** `Link.field`, Module contract kind + `validate` findings, relative Surfaces/Mirrors; convert `examples/kitchen-sink` pieces to placements and compare line counts/diagnostics.
5. **RFC:** open on foldkit-plus with the spike, the before/after diff, and the upstream question (§11.8).
