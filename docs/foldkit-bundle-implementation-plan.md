# foldkit-bundle: Detailed Design & Implementation Plan (for foldkit-plus)

Drafted 2026-09-16. Expands [foldkit-bundle-proposal.md](foldkit-bundle-proposal.md) into an API specification and a phased implementation plan that follows foldkit-plus repository conventions (read from the repo: `AGENTS.md`, `package.json`, `packages/mirror` layout, `packages/surface/src/index.ts`, `docs/releases.md`, `docs/rfc.md`).

> **Version note:** foldkit-plus currently pins `foldkit@0.158.2` and `effect@4.0.0-rc.112`. Core APIs referenced below were verified in `foldkit@0.160.0` type declarations. **Phase 0 re-verifies each against 0.158.2** (or bumps the workspace).

Legend: **[V]** verified in foldkit 0.160.0 d.ts or foldkit-plus source · **[U]** to verify in Phase 0.

---

# Part A: Design specification

## A1. Goals and non-goals
**Goals**
1. **Define once:** a child's Model, Message, OutMessage, init, update, subscriptions, resources, view and helpers form one typed value.
2. **Place anywhere:** one Link value lifts all parts into any parent, compiling to Foldkit's existing lifts.
3. **Place many:** keyed collections with correct per-item routing and lifecycles.
4. **Complete by construction:** forgetting to wire a placement's Message, subscriptions or resources is a type error (and a runtime finding).
5. **Plus-native:** placements are Module contracts (ownership validated), expose re-rooted Surfaces, and interoperate with Agent/Mirror.
6. **Zero runtime:** no new effect category, store, reducer or render path.

**Non-goals**
- Replacing `Update.foldChild`, `h.submodel`, `Subscription.lift`, `ManagedResource.lift`: bundles *call* them.
- Implicit registration or discovery; a DI container; hidden child state.
- Making bundles mandatory: hand-wired Submodels remain first-class.

## A2. Packages (mirrors the repo's `mixins` / `mixins-surface` split)
| Package | Depends on | Contents |
|---|---|---|
| **`foldkit-bundle`** | `foldkit`, `effect` (peers) | `Bundle`, `Link`, `Placed`, `Collection`, `Assembly`, `Bundle.application` |
| **`foldkit-bundle-surface`** | `foldkit-bundle`, `foldkit-surface` | `Link.field`, bundle Module contracts, relative Surfaces, Mirror/Agent helpers |

Rationale: core stays usable in vanilla Foldkit (dual-support principle); ownership/Surface features live beside `foldkit-surface` like `mixins-surface`.

## A3. Core types

### A3.1 Bundle definition
```ts
// packages/bundle/src/bundle.ts
export interface BundleSpec<
  Name extends string,
  Args,
  Model,
  Message extends { readonly _tag: string },
  OutMessage = never,
  R = never,            // requirements of Commands returned by init/update/helpers
  S = never,            // services required by subscriptions
  ViewInputs = void,
> {
  readonly name: Name
  readonly Model: Schema.Codec<Model, unknown, never, never>
  readonly Message: Schema.Codec<Message, unknown, never, never>       // tagged union
  readonly OutMessage?: Schema.Codec<OutMessage, unknown, never, never> // optional; enables agent/docs introspection
  readonly init: (args: Args) => Update.Return<Model, Message, R>        // [V] Update.Return
  readonly update: (model: Model, message: Message, args: Args) =>
    [OutMessage] extends [never]
      ? Update.Return<Model, Message, R>
      : Update.ReturnWithOutMessage<Model, Message, OutMessage, R>       // [V]
  readonly subscriptions?: (args: Args) => Subscriptions<Model, Message, S>          // [V] Subscription.make result
  readonly resources?: (args: Args) => ResourceEntries<Model, Message>               // [V] ManagedResource.make result
  readonly view?: Submodel.View<Model, Message, ViewInputs>                           // [V] SubmodelView
  readonly helpers?: Readonly<Record<string, (model: Model, ...input: any[]) => Update.Return<Model, Message, R> | Update.ReturnWithOutMessage<Model, Message, OutMessage, R>>>
}

export interface Bundle<…same params…> extends BundleSpec<…> {
  readonly [BundleTypeId]: BundleTypeId
  at<Parent, ParentMessage>(link: Link<Parent, ParentMessage, Model, Message>, ...args: ArgsParam<Args>): Placed<…>
  each<Parent, ParentMessage, Key>(link: CollectionLink<Parent, ParentMessage, Key, Model, Message>, options: EachOptions<Key, Model, Args>): PlacedCollection<…>
}

export const Bundle: {
  make<const Spec extends BundleSpec<any, any, any, any, any, any, any, any>>(spec: Spec): BundleOf<Spec>
  fromParts<…>(parts: PartsLike, spec: Omit<BundleSpec<…>, "update" | "view"> & Partial<…>): Bundle<…>  // wraps @foldkit/ui `create()` bundles
}
```
**Decisions**
- `init` returns `Update.Return` (not just Model) so a child can start Commands on placement (e.g. load initial data).
- `args` is threaded to `update` as a **3rd parameter**, so behavior can depend on config without closures in Model. `ArgsParam<void>` = no args.
- `OutMessage` is inferred from `update`'s return type when the Schema is omitted; the Schema is optional metadata.
- `helpers` are *programmatic entry points* (like `Listbox.open/close/selectItem` [V]) lifted to parent `Step`s.
- `const` type parameter on `make` preserves literal `name` for namespacing and contract labels.

### A3.2 Link
```ts
// packages/bundle/src/link.ts
export interface Link<Parent, ParentMessage, Child, ChildMessage> {
  readonly [LinkTypeId]: LinkTypeId
  readonly read: (parent: Parent) => Option.Option<Child>
  readonly write: (parent: Parent, child: Child) => Parent
  readonly toParentMessage: (message: ChildMessage) => ParentMessage
  readonly fromParentMessage: (message: ParentMessage) => Option.Option<ChildMessage>   // needed for routing
  readonly when?: (parent: Parent) => boolean                                          // gates subscriptions/resources
  readonly path: readonly string[]                                                     // diagnostics, slotId, contracts
}

export const Link: {
  make<Parent, ParentMessage, Child, ChildMessage>(config: {
    read: (parent: Parent) => Option.Option<Child> | Child     // plain Child auto-wrapped in Some
    write: (parent: Parent, child: Child) => Parent
    wrapper: Wrapper<ParentMessage, ChildMessage>             // from Link.wrapper, provides to/from
    when?: (parent: Parent) => boolean
    path?: readonly string[]
  }): Link<Parent, ParentMessage, Child, ChildMessage>

  /** Parent Message variant `Tag({ message: Child.Message })` + to/from functions. */
  wrapper<const Tag extends string, ChildMessage>(tag: Tag, childMessage: Schema.Codec<ChildMessage, unknown, never, never>): Wrapper<…> & {
    readonly Schema: Schema.Codec<{ readonly _tag: Tag; readonly message: ChildMessage }, …>   // add to parent Message union
    readonly make: (message: ChildMessage) => { _tag: Tag; message: ChildMessage }
  }

  field<Parent, K extends keyof Parent & string>(key: K, wrapper: Wrapper<…>): Link<…>   // vanilla shorthand for a struct field
  optional<…>(key, wrapper): Link<…>                                                       // field holding Option<Child>
  compose<…>(outer: Link<A, AM, B, BM>, inner: Link<B, BM, C, CM>): Link<A, AM, C, CM>     // nested placements
}
```
**Decisions**
- **`fromParentMessage` is required** because assembly-level routing must recognize a placement's Messages. `Link.wrapper` derives both directions from the tag, so users never write it.
- **Wrapper Schema construction** must match Foldkit's Message conventions (Foldkit Message unions are Schema tagged structs with `.match` [V: `Message.match` used in d.ts examples]; exact constructor helper [U]).
- **`Link.compose`** supports a bundle inside a bundle (Listbox inside a Table filter) without repeating lenses.
- `Link.field` covers most cases without foldkit-surface; `foldkit-bundle-surface` adds `Link.ref(FieldRef)` using the ref's Optic [V: `ModelRef.get/set/optic/dependency`].

### A3.3 Placed
```ts
// packages/bundle/src/placed.ts
export interface Placed<Name, Parent, ParentMessage, Model, Message, OutMessage, R, S, ViewInputs> {
  readonly [PlacedTypeId]: PlacedTypeId
  readonly name: Name
  readonly link: Link<Parent, ParentMessage, Model, Message>
  readonly Message: Schema.Codec<ParentWrapped<Message>, …>            // the wrapper variant to include in the parent union
  readonly init: Update.Step<Parent, ParentMessage, R>                  // writes child init, lifts its Commands
  readonly update: (parent: Parent, message: ParentMessage) => Option.Option<Update.Return<Parent, ParentMessage, R>>
  readonly fold: Update.Fold<Parent, ParentMessage, Message, R>        // [V] raw foldChild result for advanced use
  readonly subscriptions: LiftedSubscriptionsRecord<Parent, ParentMessage, S>   // [V] Subscription.lift output, keys namespaced
  readonly resources: LiftedResourcesRecord<Parent, ParentMessage>              // [V] ManagedResource.lift output, keys namespaced
  readonly view: (parent: Parent, h: HtmlBuilder<ParentMessage>, ...inputs: InputsParam<ViewInputs>) => Html   // via h.submodel [V]
  readonly helpers: LiftedHelpers<Parent, ParentMessage, R>             // each helper → Step<Parent, ParentMessage, R>
}

export interface PlaceOptions<Parent, ParentMessage, OutMessage, R2> {
  /** Handle the child's OutMessage in parent terms (compiles to foldChild's foldOutMessage [V]). */
  readonly onOut?: (out: OutMessage, context: Update.FoldContext<…>) => Update.Step<Parent, ParentMessage, R2>
  /** Override namespacing prefix for subscription/resource keys (default `${name}@${path.join(".")}`). */
  readonly key?: string
}
```
**Compilation (exactly what `at` builds)**
| Placed part | Built from |
|---|---|
| `fold` | `Update.foldChild({ update: (child, msg) => spec.update(child, msg, args), read: link.read, write: link.write, toParentMessage: link.toParentMessage, foldOutMessage: options.onOut })` [V] |
| `update` | `link.fromParentMessage(message)` → `Some(fold(parent, childMessage))`, else `None` |
| `init` | `(parent) => lift(spec.init(args))`: write child Model via `link.write`, map Commands with `Command.mapMessage(link.toParentMessage)` [V `mapMessage`] |
| `subscriptions` | `Subscription.lift(spec.subscriptions(args))({ toChildModel, toParentMessage, when })` [V]. `toChildModel` must return a Child (not Option): use gate `when = parent => isSome(link.read(parent)) && (link.when?.(parent) ?? true)` and `toChildModel = parent => getOrThrow(link.read(parent))` **only if** Foldkit evaluates the gate before `toChildModel` [U: GatedDependencies evaluation order]; otherwise implement a thin custom entry that short-circuits |
| `resources` | `ManagedResource.lift(spec.resources(args))({ toChildModel: parent => link.when?.(parent) === false ? None : link.read(parent), toParentMessage })` [V Option-based] |
| `view` | `h.submodel({ slotId: key, model: getOrThrow(link.read(parent)), view: spec.view, toParentMessage, viewInputs })` [V `SubmodelConfig`]; returns `null` (empty Html [V `Html = VNode \| null`]) when read is `None` |
| `helpers` | `(parent, ...input) => foldChildStep-like`: read child, run helper, write back, lift Commands, route OutMessage through `onOut` [V `foldChildStep` exists] |
| key namespacing | record keys rewritten to `${prefix}/${key}` so `aggregate`'s duplicate check [V] stays meaningful across multiple placements of the same bundle |

### A3.4 Collections
```ts
export interface CollectionLink<Parent, ParentMessage, Key, Child, ChildMessage> {
  readonly read: (parent: Parent) => ReadonlyMap<Key, Child> | HashMap.HashMap<Key, Child> | Readonly<Record<string, Child>>
  readonly write: (parent: Parent, key: Key, child: Option.Option<Child>) => Parent   // None = remove
  readonly toParentMessage: (key: Key, message: ChildMessage) => ParentMessage
  readonly fromParentMessage: (message: ParentMessage) => Option.Option<readonly [Key, ChildMessage]>
  readonly keySchema: Schema.Codec<Key, string | number, never, never>
  readonly when?: (parent: Parent, key: Key) => boolean
  readonly path: readonly string[]
}
Link.collection(config) / Link.keyedWrapper(tag, keySchema, childMessage)

export interface PlacedCollection<…> {
  readonly Message: …                         // `Tag({ key, message })`
  readonly add: (key: Key, ...args) => Update.Step<Parent, ParentMessage, R>     // init + write
  readonly remove: (key: Key) => Update.Step<Parent, ParentMessage, never>        // write None (lifecycles end via Model)
  readonly update: (parent, message) => Option<Return>   // unknown key → Some({ model: parent }) + dev warning hook
  readonly view: (parent, h, key, ...inputs) => Html
  readonly viewAll: (parent, h, render: (key, childView: Html) => Html) => ReadonlyArray<Html>
  readonly subscriptions: …                    // see lifecycle below
  readonly resources: …                        // v1: per-item resources unsupported at type level (see below)
  readonly helpers: …                          // (parent, key, ...input) → Step
}
```
**Subscription lifecycle for collections**
- **v1 (correct, simple):** for each child subscription key `k`, one parent entry:
  - `modelToDependencies(parent)` = sorted array of `[keyString, childDeps]` for keys passing `when`.
  - `dependenciesToStream(deps)` = `Stream.mergeAll(deps.map(([key, d]) => childEntry.dependenciesToStream(d).pipe(Stream.map(m => toParentMessage(key, m)))), { concurrency: "unbounded" })`.
  - Any dependency change restarts all child streams for that entry (Foldkit's default restart semantics [V]).
- **v2 (per-key keep-alive):** use `keepAliveEquivalence` + `readDependencies` [V both exist] to keep one long-lived stream that diffs the key set on each Model change and starts/stops per-key fibers (`FiberMap` keyed by key [V exists in effect core]). **Spike required** [U: whether `readDependencies` reflects the latest Model on each change without a change signal; may need a `SubscriptionRef` of dependencies].
- Child entries that are themselves keep-alive: preserve their equivalence per key in v2.

**Resources for collections**
- Foldkit ManagedResource entries are static keyed records with per-entry requirements [V]; they can't express N dynamic resources.
- **v1:** `each` rejects bundles with `resources` at the type level (a readable `Invalid<"Bundle.each does not support resources yet; see docs/bundle.md#collections">` [pattern V: foldkit-plus `Invalid` type exists]).
- **v2 options:**
  - (a) `Bundle.pool`: one parent resource holding an `RcMap<Key, Value>` [V effect core], plus per-key acquire/release Subscriptions.
  - (b) Upstream proposal `ManagedResource.each` to Foldkit.

  Decide after the v2 spike.

### A3.5 Assembly and completeness
```ts
// packages/bundle/src/assembly.ts
export const Bundle.assemble = <const Ps extends ReadonlyArray<AnyPlaced<Parent, ParentMessage>>>(placements: Ps) => Assembly<Ps>

export interface Assembly<Ps> {
  readonly placements: Ps
  readonly route: (parent: Parent, message: ParentMessage) => Option.Option<Update.Return<Parent, ParentMessage, RequirementsOf<Ps>>>
  readonly init: Update.Step<Parent, ParentMessage, RequirementsOf<Ps>>     // Update.combine of placement inits [V combine]
  readonly subscriptions: MergedRecord<SubscriptionsOf<Ps>>                   // branded
  readonly resources: MergedRecord<ResourcesOf<Ps>>                           // branded
  readonly Messages: readonly Schema.Codec<…>[]                               // wrapper variants to spread into the parent union
}

export const Bundle.application = <…>(config: {
  Model, Message, init, update, view,
  assembly: Assembly<Ps>,
  subscriptions: Includes<SubscriptionsOf<Ps>>,     // type error if assembly.subscriptions not aggregated in
  resources: Includes<ResourcesOf<Ps>>,
  …runtime options
}) => Parameters<typeof Runtime.makeApplication>[0]   // [V makeApplication exists; exact config type U]
```
**Completeness checks**
- **Type level:**
  - the parent `Message` type must include every `Placed.Message` type (`Subset` check, same technique as Surface [V: REVISION_PLAN "Subset check"])
  - `subscriptions`/`resources` must structurally include the assembly's branded records
  - errors surface as named `Invalid<…>` messages at the offending property
- **Runtime (dev):** `Bundle.application` asserts that every namespaced key exists in the supplied records and that every placement `Message` tag decodes against the parent Message Schema; throws at startup with the placement name (aligns with `aggregate` "fails loudly at startup" [V]).
- **Update routing precedence:** assembly `route` first, then the parent's own update. Documented: a parent must not also handle placement wrapper tags (caught by the Message ownership finding in bundle-surface).

## A4. `foldkit-bundle-surface` (plus integration)
| API | Behavior | Builds on |
|---|---|---|
| `Link.ref(fieldRef, wrapper)` | Link from a `FieldRef`/`ModelRef`: `read = Some(ref.get(parent))`, `write = ref.set`, `path = ref.dependency` | `ModelRef` [V] |
| `BundleContract.of(placed)` / `placed.contract` | `Contract { kind: "bundle", name: \`${name}@${path}\`, owner: app.owner, owns: [path], observes: [], messages: [wrapperTag], metadata: [] }` so `Module.make(App, [placedA, …])` accepts placements | `Contract`, `ModuleItem` accepts `{ contract }` [V] |
| Module findings | Existing rules apply unchanged: `ownership-overlap` (two placements on one path, or a placement vs Sync/Remote), `unknown-path`, `unknown-message`, `duplicate-name` [V rule names] | `Module.validate` [V] |
| New finding (proposal to surface) | `bundle-unwired`: `Module.validate(module, { subscriptions, resources })` optional second arg reports placements whose namespaced keys are missing | extends `Finding.rule` union |
| Relative Surfaces | Bundle spec gains `surfaces?: (B: BundleScope<Model, Message>) => Record<string, Surface<Model, …>>`; `placed.surfaces` re-roots each via the link: projection composed with the ref optic; Messages mapped with the wrapper | `Projection`, `Surface.make` [V exist]; projection re-rooting helper [U: whether Projection exposes composition over an Optic; else add `Projection.at(ref, projection)` to surface] |
| Mirrors | Bundle spec gains `mirrorable?: { url?: fields, kv?: fields }` (relative refs + codecs); `placed.mirrors.url(prefix)` / `.kv(key)` produce `Mirror` declarations with namespaced params | `foldkit-mirror` API [U: exact Mirror.url config] |
| Agent | `placed.surfaces` pass straight into `Agent…make({ context, messages })`; tool names prefixed by placement name | `Agent` [V exists] |
| SurfaceView | Bundle `view` may be a `SurfaceView` (mixins-surface); `placed.view` preserves Slots | mixins-surface [V exists] |

## A5. Worked example (end to end, vanilla)
```ts
// media-query.ts
export const MediaQuery = Bundle.make({
  name: "MediaQuery",
  Model: Schema.Struct({ matches: Schema.Boolean }),
  Message: MediaQueryMessage,                                  // Changed({ matches })
  init: (_: { query: string }) => ({ model: { matches: false } }),
  update: (model, message) => MediaQueryMessage.match(message, { Changed: ({ matches }) => ({ model: { matches } }) }),
  subscriptions: ({ query }) => Subscription.make<{ matches: boolean }, MediaQueryMessage>()(() => ({
    changes: Subscription.persistent(matchMediaStream(query)),
  })),
})

// app.ts
const GotDark = Link.wrapper("GotDarkMessage", MediaQueryMessage)
const Dark = MediaQuery.at(Link.field("prefersDark", GotDark), { query: "(prefers-color-scheme: dark)" })
const Assembly = Bundle.assemble([Dark])

const Message = Schema.Union([ToggledTheme, GotDark.Schema])        // include wrapper (type-checked)
const app = Bundle.application({
  Model, Message, assembly: Assembly,
  init: () => Update.combine(initialModel, [Assembly.init]),          // [V combine data-first]
  update: (model, message) => Option.getOrElse(Assembly.route(model, message), () => ownUpdate(model, message)),
  subscriptions: Subscription.aggregate<Model, Message>()(ownSubscriptions, Assembly.subscriptions),
  resources: ManagedResource.aggregate<Model, Message>()(Assembly.resources),
  view: (model, h) => h.div([], [model.prefersDark.matches ? "dark" : "light"]),
})
```

---

# Part B: Implementation plan

## B0. Conventions to follow (from foldkit-plus `AGENTS.md` and repo layout)
- **Package layout** (copy `packages/mirror`): `src/`, `test/` (`*.test.ts`, `*.test-d.ts`, `readme.test-d.ts`), `tsconfig.json` (composite, `paths` to sibling sources), `tsconfig.build.json` (excludes test), `tsdown.config.ts` (`neverBundle: effect, foldkit, …`), `package.json` (`sideEffects: false`, ESM exports to `dist/index.mjs`, peers `effect ^4.0.0-rc.112`, `foldkit ^0.158.2`), `README.md`, `LICENSE`.
- **Workspace registration:**
  - root `tsconfig.json` references
  - `vitest.config.ts` aliases (`foldkit-bundle` → `packages/bundle/src/index.ts`)
  - `docs/releases.md` matrix rows
  - root `demo` script gains the new example
- **Process:**
  - Commit small and often; after each commit re-run `pnpm typecheck`, `pnpm test`, `pnpm build`.
  - Review for correctness, edge cases, synergy, DX, comments, tests, security and performance.
  - De-slop (no dead abstractions, no unused exports, no restating comments).
- **Tests must be able to fail:** no redundant guards; table-driven cases; type tests for DX (`.test-d.ts`); README snippets type-checked (`readme.test-d.ts`).
- **Examples are acceptance tests:** an example prints a transcript pinned by its test; CI runs `pnpm demo`.
- **README order:** what it is → ownership boundary → mental model → install → 60-second example → concepts → workflows → integrations → failure semantics → advanced → limits.
- **Declaration portability:** publish nameable public types (`Bundle`, `Placed`, `Link`, `Assembly`) to avoid TS2742 (known issue in `docs/improvements.md`).

## B1. Phase 0: Verification spike (1–2 days)
**Goal:** de-risk every [U] before committing to signatures.

| # | Task | Output |
|---|---|---|
| 0.1 | Branch `feat/bundle-spike`; decide foldkit version: verify APIs in `0.158.2`, else bump workspace to `0.160.0` in a separate PR | Version decision note |
| 0.2 | Verify `Update.foldChild` / `foldChildStep` / `FoldContext` / `combine` shapes and OutMessage variants | Notes + minimal compile file |
| 0.3 | Verify `Subscription.lift` gating: is `when` evaluated before `toChildModel`? What are `GatedDependencies` semantics when gated off? | Decision: use gate+`getOrThrow` vs custom entry |
| 0.4 | Verify `ManagedResource.lift` with `Option` child and key namespacing (can lifted record keys be renamed safely?) | Notes |
| 0.5 | Verify `h.submodel` config (`slotId`, `viewInputs`) and empty `Html` when child absent | Notes |
| 0.6 | Verify Foldkit Message union construction helpers for building a wrapper variant Schema generically | Wrapper implementation choice |
| 0.7 | Prototype `keepAliveEquivalence` + `readDependencies` per-key streams (collections v2 feasibility) | Go/no-go for v2 in Phase 3 |
| 0.8 | Check `Runtime.makeApplication` config type for `Bundle.application` wrapping | Type strategy |
| 0.9 | Hand-wire `MediaQuery`, `Pagination`, and `Listbox.create` placement in a scratch example; record Message logs | Baseline for parity tests |

**Exit criteria:** every [U] resolved or explicitly deferred; spike branch not merged; this plan updated.

## B2. Phase 1: `foldkit-bundle` core: single placement (4–6 days)
**Files**
```
packages/bundle/
  package.json  tsconfig.json  tsconfig.build.json  tsdown.config.ts  README.md  LICENSE
  src/index.ts          public exports only
  src/bundle.ts         Bundle.make, Bundle.fromParts, BundleTypeId, types
  src/link.ts           Link.make, Link.field, Link.optional, Link.compose, Link.wrapper
  src/placed.ts         at(): fold, update, init, subscriptions, resources, view, helpers
  src/namespace.ts      record key namespacing (subscriptions/resources)
  src/invalid.ts        Invalid<…> readable type errors
  test/link.test.ts
  test/placed.update.test.ts
  test/placed.effects.test.ts        subscriptions/resources lifting with fake streams
  test/placed.view.test.ts           Scene-based view tests (foldkit/test)
  test/parity.test.ts                hand-wired vs placed Message logs (from 0.9)
  test/types.test-d.ts               inference, OutMessage typing, args typing, errors land at mistake
  test/readme.test-d.ts
```
**Commit sequence**
1. `bundle: scaffold package` (manifest, tsconfigs, tsdown, empty index; register in root tsconfig, vitest alias)
2. `bundle: Link.wrapper and Link.make` + tests (to/from round-trip, unknown tag → None, Schema decode of wrapper)
3. `bundle: Link.field, optional, compose` + table tests (absent child, nested write-through)
4. `bundle: Bundle.make and fromParts` + type tests (literal name, OutMessage inference, args void vs required)
5. `bundle: Placed.update and fold via Update.foldChild` + tests (routing, OutMessage onOut, Commands lifted)
6. `bundle: Placed.init and helpers` + tests (init Commands mapped, helper Steps with OutMessage)
7. `bundle: Placed.subscriptions with gating and namespacing` + tests (absent child emits nothing; two placements of same bundle don't collide; `when` false stops stream)
8. `bundle: Placed.resources` + tests (acquire on Some, release on None/when false, onAcquired lifted)
9. `bundle: Placed.view via h.submodel` + Scene tests (absent → empty, viewInputs typed)
10. `bundle: parity tests against hand-wired baselines`
11. `bundle: README (sixty-second MediaQuery example) + readme.test-d.ts`

**Acceptance**
- Parity: identical Message logs and Models for MediaQuery, Pagination, Listbox(`fromParts`) vs hand-wired.
- Type tests prove: wrong child Message in `toParentMessage` fails at the link; missing required args fails at `at`; OutMessage handler receives the exact child OutMessage type.
- No `any` in public types; `tsc -b --force` clean; declaration emit works in a consumer example (no TS2742).

## B3. Phase 2: Assembly & completeness (3–4 days)
**Files:** `src/assembly.ts`, `src/application.ts`, `test/assembly.test.ts`, `test/application.test.ts`, `test/completeness.test-d.ts`.

**Commits**
1. `bundle: Bundle.assemble route/init/merged records` + tests (routing precedence, multiple placements, init Commands order via `Update.combine`)
2. `bundle: branded records and Includes type` + negative type tests (omitted `Assembly.subscriptions` → named `Invalid` error)
3. `bundle: parent Message Subset check` + negative type tests (wrapper variant missing from union)
4. `bundle: Bundle.application runtime dev assertions` + tests (throws naming the placement; production no-op flag if needed)
5. `bundle: README workflows section (assemble, application)`

**Acceptance**
- The three wiring mistakes (missing Message variant, subscriptions, resources) each fail at compile time with a readable message at the mistake, and at startup if types are bypassed (`as any`).
- Performance check: `tsc` time for a 30-placement fixture recorded in `docs/benchmarks.md` (type-instantiation depth OK).

## B4. Phase 3: Collections (5–8 days)
**Files:** `src/collection.ts`, `src/collectionLink.ts`, `test/collection.*.test.ts`, `test/collection.types.test-d.ts`, `bench/collection.bench.ts`.

**Commits**
1. `bundle: Link.collection and keyedWrapper` (Record, ReadonlyMap, HashMap backings; key Schema codec)
2. `bundle: each update/add/remove/helpers` + tests (unknown key, remove then late message ignored, add existing key semantics)
3. `bundle: each view/viewAll with key-derived slotIds` + Scene tests
4. `bundle: each subscriptions v1 (merged streams, restart on change)` + tests (item added starts stream, removed stops, gate per key)
5. `bundle: type-level rejection of resources in each` + test-d
6. *(if 0.7 go)* `bundle: each subscriptions v2 per-key keep-alive` using `FiberMap` + tests proving untouched keys aren't restarted + bench
7. `bundle: collection docs section`

**Acceptance**
- 1,000-item collection: routing O(1) per message (map lookup); v2 adding one item restarts **0** other item streams (bench + test).
- Clear docs on v1 restart semantics if v2 is deferred.

## B5. Phase 4: `foldkit-bundle-surface` (4–6 days)
**Files**
```
packages/bundle-surface/
  src/index.ts
  src/ref.ts          Link.ref (FieldRef/ModelRef)
  src/contract.ts     placed.contract, BundleContract.of
  src/surfaces.ts     relative Surfaces + re-rooting
  src/mirrors.ts      relative mirrorable fields → Mirror declarations
  test/ref.test.ts  test/contract.test.ts  test/module.test.ts  test/surfaces.test.ts  test/mirrors.test.ts  test/types.test-d.ts  test/readme.test-d.ts
```
**Commits**
1. `bundle-surface: scaffold` (deps `foldkit-bundle`, `foldkit-surface`; tsconfig references)
2. `bundle-surface: Link.ref from FieldRef` + tests (foreign-app ref rejected via owner token)
3. `bundle-surface: bundle contracts in Module` + tests (`ownership-overlap` between two placements and vs a Sync contract; manifest shows owner `bundle:Name@path`)
4. `surface: Projection re-rooting helper` (**small PR to foldkit-surface** if missing [U]) + tests
5. `bundle-surface: relative Surfaces re-rooted per placement` + tests (agent tool per placement; Message subset wraps correctly)
6. `bundle-surface: optional bundle-unwired finding` (either in `Module.validate` options or `BundleModule.validate`) + tests
7. `bundle-surface: mirrors (url prefix / kv key)` + tests (two placements don't collide in URL)
8. `bundle-surface: README (plus path; ownership story first)`

**Acceptance**
- `Module.validate` catches double placement on a path and placement-vs-Sync overlap with existing rule names.
- Two placed tables on one page expose two distinct agent tool sets and two URL namespaces.

## B6. Phase 5: Examples, docs, skill, release (3–5 days)
| Task | Details |
|---|---|
| `examples/bundle` | A settings page: MediaQuery (subscription), Pagination (state + reflect), Uploads collection (keyed), Listbox via `fromParts`, OutMessage handling; `demo` prints a transcript; test pins it; added to root `demo` script |
| kitchen-sink migration | Convert one hand-wired Submodel in `examples/kitchen-sink` to a placement; show diff (lines, wiring) in PR |
| `docs/bundle.md` guide | Mental model diagram: `Bundle (what) + Link (where) = Placed (lifted native parts)` · ownership (Bundle = ownership unit; Surface = access) · collections lifecycle · completeness · limits (resources in collections) |
| `docs/README.md` index | Add bundle guide |
| `docs/releases.md` | Rows for `foldkit-bundle` 0.1.0, `foldkit-bundle-surface` 0.1.0 |
| `skills/foldkit-plus` | Add reference: when to use a Bundle vs hand-wiring; the three wiring mistakes; collection recipes |
| Root README | "Which package do I need?" row: *Package a feature once and place it anywhere or many times* → `foldkit-bundle` |
| `docs/rfc.md` | Section: Bundles as the missing packaging primitive, and the upstream question (Link + `at` in core?) |
| Release | `pnpm check` green → version 0.1.0 → `pnpm release` (pnpm only, per releases doc) |

## B7. Phase 6: Upstream & ecosystem (ongoing)
1. **Foldkit core proposal:** `Link` + `Bundle.at` as a small core addition (it only packages core lifts); `ManagedResource.each` if collections v2 shows need. Bring parity tests and line-count diffs.
2. **`@foldkit/ui` alignment:** propose that `create()` bundles export `Model/Message/init` alongside, making `fromParts` unnecessary.
3. **foldkit-plus packages returning Bundles:** `Remote.bundle(…)`, `Sync.bundle(…)`, so infrastructure is placeable like features (addresses "cross-package API consistency" in `docs/improvements.md`).
4. **pocket adoption:** `@pocket/foldkit-primitives` and TanStack-backed components (table/virtual/range/hotkeys/charts/forms) ship as Bundles.

## B8. Testing matrix
| Layer | Tooling | What it proves |
|---|---|---|
| Pure units | vitest | Link round-trips, routing, key namespacing, collection add/remove |
| Update semantics | vitest + foldkit `Update` | Commands lifted, OutMessage mapped, helpers |
| Effects | vitest with `Stream` fakes, `TestClock` | Subscription gating/restart/keep-alive; resource acquire/release |
| Views | `foldkit/test` Scene/Story [V exports exist] | h.submodel embedding, absent child, collection slotIds |
| Parity | recorded Message logs | Bundles == hand-wired |
| Types | `*.test-d.ts` (`expectTypeOf`) | inference, `Invalid` errors land at mistakes, no `any` leaks |
| Docs | `readme.test-d.ts` | README code compiles |
| Examples | `pnpm demo` transcripts | end-to-end acceptance |
| Perf | `vitest bench`, `tsc` timing | collection routing and restarts; type-check cost |

## B9. Risks & mitigations
| Risk | Mitigation |
|---|---|
| Type complexity/slow `tsc` with many placements | Keep generics shallow; branded records instead of deep conditional merges; benchmark in Phase 2; provide explicit `Placed<…>` annotations pattern |
| Foldkit API drift (0.158 → 0.160+) | Phase 0 pin decision; parity tests catch behavior changes; thin compilation layer isolates changes |
| `Subscription.lift` gating semantics don't support Option children | Custom entry wrapper (small) instead of lift, decided in Phase 0 |
| Per-key subscription keep-alive infeasible | Ship v1 restart semantics with docs; propose upstream support |
| Overlap with Surface concepts confuses users | Docs lead with ownership vs access contrast; bundle-surface keeps contracts explicit |
| Wrapper Message naming conventions differ across apps | `Link.wrapper(tag, …)` explicit; lint/docs recommend `Got<Name>Message` (existing Foldkit convention [V in d.ts docs]) |
| Scope creep (Flags/Ports/Mounts in bundles) | Out of scope for 0.1; collect use cases (open question) |

## B10. Timeline summary
| Phase | Duration | Deliverable |
|---|---|---|
| 0 Verification spike | 1–2 d | Resolved [U] list, version decision, baselines |
| 1 Core single placement | 4–6 d | `foldkit-bundle` at/Link/Bundle with parity |
| 2 Assembly & completeness | 3–4 d | compile-time wiring guarantees |
| 3 Collections | 5–8 d | `each` v1 (+ v2 if feasible) |
| 4 bundle-surface | 4–6 d | Module contracts, Surfaces, Mirrors |
| 5 Examples/docs/release | 3–5 d | example, guide, skill, 0.1.0 release |
| **Total** | **~20–31 working days** | |

## B11. Definition of done (0.1.0)
- `pnpm check` green (format, typecheck, tests, demo, pack).
- Parity tests pass for three baseline Submodels; kitchen-sink migration merged.
- All three wiring mistakes fail at compile time with readable errors.
- Collections v1 shipped and documented (v2 shipped or explicitly deferred with an issue).
- `Module.validate` recognizes bundle contracts; two placements of one bundle are independent (Surfaces, Mirrors, keys).
- READMEs follow the repo standard; guide, skill reference, releases matrix and root README updated.

---

# Part C: Bundles as Foldkit's primitives library (the solid-primitives equivalent)

## C1. Positioning
**"Foldkit Bundles: primitives for the Elm architecture."** Use the solid-primitives catalogue (~80 packages, mapped in [foldkit-primitives.md](foldkit-primitives.md)) as a *checklist*, not a port. The library ships the right Foldkit construct per kind, with **Bundles as the packaging unit for anything stateful**, under one naming/docs convention.

## C2. What becomes a Bundle (and what doesn't)
| Primitive kind | Solid examples | Foldkit form | Bundle? |
|---|---|---|---|
| **Stateful + effectful** | `createPagination`, `createGeolocation`, `createWebsocket`, `createMediaQuery`, tween/spring, history (undo) | Model + Message + update + subscriptions/resources | **Yes**, the core use case |
| **Collections of stateful items** | lists of uploads, sockets, timers | keyed children | **Yes** (`Bundle.each`) |
| **Stream source only** | `createEventListener`, page-visibility, connectivity | Subscription entry mapped to the parent's Message | **Optional.** Ship as an entry; also offer a tiny bundle when a stored fact is useful (e.g. `Online` keeps `online: boolean`) |
| **Element-scoped** | resize/intersection/mutation observers, gestures, autofocus, input-mask | `Mount` (+ foldkit-plus `Behavior`) | **No.** Attaches in views, not Model slots |
| **One-shot actions** | clipboard, share, fullscreen request, script-loader | `Command.define` | **No** |
| **Pure helpers** | date, masonry math, range, i18n formatting | functions | **No** |

Every item, bundle or not, ships docs with a **"Solid primitive → Foldkit"** side-by-side example.

## C3. Experience comparison (be honest in docs)
| | Solid | Foldkit today (hand-wired) | Foldkit with Bundles |
|---|---|---|---|
| Use a media query | `const m = createMediaQuery(q)` (1 line) | Model field, wrapper Message, `foldChild`, `Subscription.lift`, aggregate (4–5 lifts) | one placement line + spreading the Assembly once per app (with C4) |
| Where state lives | hidden signal | Model | Model |
| Replay / time travel | no | yes | yes |
| Messages visible in DevTools | no | yes | yes |
| Wiring mistakes | n/a | silent | **compile-time errors** |
| Agents / URL mirror per instance | manual | manual | **derived** (bundle-surface) |

## C4. Ergonomic additions to the core design (fold into Phases 1–2)
| Addition | API sketch | Removes | Phase |
|---|---|---|---|
| **Default wrapper naming** | `Link.field("prefersDark")` → auto `GotPrefersDarkMessage` wrapper (Foldkit's `Got…Message` convention); explicit `Link.wrapper` still available | writing the wrapper tag | 1 |
| **Model fields from placements** | `Bundle.modelFields(Assembly)` → `{ prefersDark: MediaQuery.Model, … }` spread into the parent `Schema.Struct` | redeclaring child fields | 2 |
| **Message union from placements** | `Schema.Union([...ownMessages, ...Assembly.Messages])` as the canonical form | forgetting wrapper variants | 2 |
| **Presets** | `MediaQuery.prefersDark`, `MediaQuery.reducedMotion`, `Timer.every("1 second")` | args boilerplate | 5b |
| **Single registration list** | `Bundle.assemble([...])` is the only list; route/init/subscriptions/resources/Messages/modelFields derive from it | parallel lists | 2 |
| **`Bundle.place` shorthand** | `Bundle.place("prefersDark", MediaQuery.prefersDark)` = field Link + auto wrapper + `at` | Link construction for the common case | 2 |

Target: typical use = **one line per placement**, plus spreading the Assembly into Model, Message, subscriptions and resources once per app.

## C5. Library packaging
- **Start in pocket:** `@pocket/foldkit-primitives` on top of `foldkit-bundle`, prioritized by the pocket admin, to validate the API with real use while Phases 0–2 land.
- **Upstream candidate:** a `foldkit-bundles` collection package in foldkit-plus (separate from the `foldkit-bundle` mechanism) once 10–15 primitives are stable.
- **Layout** (subpath per primitive, tree-shakable):
```
foldkit-bundles/
  media/     MediaQuery (+ presets), Breakpoints                        [bundle]
  net/       Online, WebSocket, Sse [bundle]; BroadcastChannel [entry + command]
  time/      Timer, Interval [bundle]; debounce/throttle [commands]
  state/     Pagination, History (undo/redo), SelectionSet, Locale      [bundle]
  motion/    Tween, Spring, Presence                                     [bundle + Mount]
  device/    Geolocation, MediaDevices, MediaStream, Permissions [bundle]; Fullscreen [command + entry]
  events/    visibility, keyboard-state, pointer, scroll                 [entries; optional bundles]
  observers/ resize, intersection, mutation, bounds                      [Mounts + Behaviors]
  dom/       autofocus, input-mask [Mounts]; clipboard, share, script-loader [commands]
  plus/      Surfaces/Mirrors for bundles (optional peer deps)
```
- **First dozen:**
  - Bundles: MediaQuery, Online, Timer, Pagination, History, WebSocket, Sse, Geolocation, Tween, Presence, Locale, SelectionSet
  - Mounts: resize, intersection
  - Commands: clipboard, share
- **Per-primitive checklist:**
  - SSR-safe `init`
  - test fakes (TestClock, fake matchMedia/observers)
  - parity test (bundle vs hand-wired)
  - `readme.test-d.ts` snippet
  - side-by-side Solid example
  - `/plus` Surface/Mirror where meaningful

## C6. Added phase: Phase 5b, primitives library seed (4–6 days; parallel, in pocket)
1. Implement C4 ergonomics if not already in Phases 1–2.
2. Build MediaQuery, Online, Timer, Pagination, History, WebSocket bundles; resize/intersection Mounts; clipboard Command.
3. Side-by-side docs page template (Solid snippet ↔ Foldkit snippet).
4. Measure wiring lines per placement vs hand-wired; publish as RFC evidence.

---

# Part D: Improving foldkit-plus `docs/rfc.md` with Bundles

The RFC ("What Foldkit Might Want to Steal from foldkit-plus", 25 sections) argues for an explicit substrate (Application descriptor, Projection, MessageSet, Surface, plus interpreters) and a strong **ownership vs observation** principle (§15). It has **no packaging primitive for ownership units**: Submodels remain a hand-wired pattern. Bundles fill that gap and pass the RFC's own tests (§1 constraints, §24 "complete descriptions").

## D1. Section-by-section changes
| RFC section | Change |
|---|---|
| **Summary** | Add Bundle to the substrate diagram as the **ownership-unit descriptor**: `Application (root machine) · Bundle (reusable child machine) · Link (placement) · Projection · MessageSet · Surface`. One sentence: "Surface describes access boundaries; Bundle describes ownership units and where they are placed." |
| **§1 Design constraint** | Use Bundles as a worked example passing each rule: optional; no hidden state (a Bundle *is* a Submodel descriptor); compiles to existing categories (`foldChild`, `h.submodel`, `Subscription.lift`, `ManagedResource.lift`); writes remain Messages. Cite the "simplifies several features at once" bar with evidence: four core lifts, `@foldkit/ui` `Bundle`/`create()`, `Sync.mount`, `Remote.subscriptions` |
| **§2 Application descriptor** | Extend: **a Bundle is to a child machine what `Application.define` is to the root.** Propose one shared shape (`Model, Message, init, update, view` + optional `subscriptions`, `resources`) so the root Application is a Bundle placed at the root; DevTools/Module/Agent treat root and children uniformly |
| **§3 Projection** | Add "re-rooting through a Link": relative Projections on a Bundle compose with the placement lens → per-instance Projections. Name the needed primitive (`Projection.at(ref, projection)`) |
| **§4 Message subsets** | Note `Link.wrapper` yields the parent-side variant; a Bundle's MessageSets lift through it and stay constructor-referenced after placement |
| **§5 Surface** | Add "Relative Surfaces": declared on a Bundle, re-rooted per placement; two placements = two Surfaces (two tables → two agent tool sets) |
| **§6 Async** | Bundles add no async vocabulary; collections use Effect (`FiberMap`) internally for per-key lifetimes, consistent with "Effect remains the async-control language" |
| **§7 What this enables** | Add **7.12 Reusable primitives without a reactive runtime** (Part C) and **7.13 Collections of machines** (keyed per-item lifecycles, today's hardest Submodel case) |
| **§8 Architecture as data** | Placements are declarations: DevTools/`Module.toMermaid` render the **ownership tree** (bundle → path → derived Surfaces) next to Message history |
| **§9 Agent** | Per-placement tool namespacing; Bundles can ship agent Surfaces |
| **§10 Mirror** | Bundle-declared mirrorable fields with placement-prefixed URL params (`?invoices.sort=…`) to avoid collisions |
| **§11 Mixins / Parts** | A Bundle's view can publish Parts/Slots; Styles attach per placement, outside state |
| **§12–13 Remote / Sync** | Suggest `Remote.bundle` / `Sync.bundle` Bundle-shaped values replacing ad hoc `mount`/`subscriptions` wiring (matches `improvements.md` "cross-package API consistency") |
| **§15 Ownership vs observation** | Add rows: `Bundle placed at P = owns P's transitions`; `Link = where ownership lives ≠ a second owner`; `two placements = two owners of two paths`. Review rule: *"Every reused ownership unit should be a Bundle; every placement should be a Module contract."* |
| **§16 Proposed architecture** | Add `Bundle · Link · Placed` beside `Submodel`, compiling down to `Command · Subscription · Mount · ManagedResource` |
| **§17 Concrete candidate API** | New subsections **Bundle**, **Link**, **Placement/Assembly**, **Collections** with the minimal API (`make`, `at`, `each`, `assemble`, `Link.field`/`wrapper`, `Bundle.place`) |
| **§18 Not to upstream** | Add: no implicit bundle registry/discovery; no DI container; no per-bundle runtime; no bundle-owned hidden state; no automatic placement from Schema alone |
| **§19 Staged plan** | Insert **Phase 1b: Bundle + Link substrate** (depends only on core lifts + Application shape); tooling phase reads placements; collections later |
| **§20 Experiments** | Add **Experiment G: Bundle-ify three Submodels** (MediaQuery, Pagination, `@foldkit/ui` Listbox via `fromParts`): parity logs, wiring line counts, compile-time wiring errors. **Experiment H: keyed collection lifecycles** (1,000 items; restarts per add) |
| **§21 Story** | "Foldkit has one state machine, and now reusable machines: define once, place anywhere, place many, still replayable." |
| **§22 Priority table** | Insert **Bundle + Link** at ~4–5 ("Strongly consider as substrate", beside Application descriptor); **Bundle collections** ~8 ("Prototype"); **Bundles primitives library** ~11 ("Official optional package after Experiment G") |
| **§23 Main risk** | Acknowledge Bundles add a noun. Mitigation: plain Submodels stay idiomatic for one-off children; recommend Bundles for **reuse** (placed ≥ 2 times or published); `Bundle.place` keeps call sites small |
| **§24 Complete descriptions** | Add to the substrate list: `Bundle: reusable complete description of a child machine (state, transitions, effect requirements)` and `Link: complete description of placement (where state lives, how Messages route)`, each supporting several interpreters (update folding, subscriptions, resources, views, Module ownership, Surfaces, Mirrors, docs) |
| **§25 Final recommendation** | Add a stress-test question: *"Can a reusable feature be packaged and placed many times without a second store, a hidden runtime, or hand-written wiring?"* |

## D2. Deliverables
- A PR to foldkit-plus `docs/rfc.md` implementing D1 (concise prose, one diagram per concept, per `AGENTS.md` "inflated prose" rules), linked to Phase 0 spike results and Experiment G evidence.
- `docs/design/bundle-DESIGN.md` (Part A of this plan, trimmed) as the design note the RFC links to; add both to `docs/README.md`.

## D3. Updated timeline
| Work | Duration |
|---|---|
| Phases 0–5 (Part B) | ~20–31 days |
| Phase 5b primitives seed (Part C) | 4–6 days (parallel, in pocket) |
| RFC update PR (Part D) | 1–2 days, after Phase 0–1 evidence |
