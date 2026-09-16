# Foldkit Primitives: a Solid Primitives Equivalent for the Elm Architecture

Drafted 2026-09-16. Sources:
- **solid-primitives:** solidjs-community/solid-primitives (1.6k★, ~80 packages)
- **Foldkit:** `foldkit@0.160.0` type declarations (subscription, managedResource, command, submodel, mount, port modules)

Related docs: [foldkit-tanstack-components-design.md](foldkit-tanstack-components-design.md), [react-in-foldkit.md](react-in-foldkit.md).

## 1. The question
Solid Primitives are small composable functions (`createEventListener`, `createMediaQuery`, `createStorage`, `createWebsocket`, `createPagination`…) that return **reactive signals** and auto-cleanup with the owner. Foldkit has no signals: state is one Model, changes are Messages through `update`, side effects are Effects. What's the equivalent?

**Answer:** not a single construct. A Solid primitive bundles *state + effects + lifecycle*. Foldkit already splits those into explicit, typed pieces, and a "Foldkit primitive" is a **small, liftable bundle of those pieces**.

## 2. Foldkit's building blocks (verified in type declarations)
| Construct | What it is (per Foldkit docs in d.ts) | Solid analogue |
|---|---|---|
| **Subscription** (`Subscription.make`) | Entries map `modelToDependencies(model)` → `dependenciesToStream(deps)`; the Stream restarts when dependencies change. `Subscription.persistent(stream)` runs for the app's lifetime regardless of Model. Helpers: `fromEvent`, `fromEventFilterMap(…PreventDefault)`, `animationFrame` (deltaTime), `aggregate` (merge records, throws on duplicate keys), **`lift`** (embed a child's subscriptions with `toChildModel`, `toParentMessage`, `when`) | `createEffect` + `onCleanup` + event listener/observer primitives |
| **ManagedResource** (`ManagedResource.make`, `.tag`) | Long-lived resource acquired/released when `modelToMaybeRequirements(model)` becomes `Some`/`None`; callbacks `onAcquired`, `onAcquireError`, `onReleased`; `tag<Value>()(key)` gives an identity with `.get` for use in Commands; **`lift`** for child Submodels (requirements `Option`-wrapped) | `createResource` / owned objects (websocket, media stream, worker) with `onCleanup` |
| **Command** (`Command.define`, `mapEffect`, `mapMessage(s)`) | One-shot Effects returned by `update`, producing Messages | Imperative calls inside handlers, `createResource` fetchers |
| **Mount** (`Mount.define`, `Mount.defineStream`) | Per-element effect: `(element, viewStateChanges) => Stream<Message>`, scope bound to element lifetime | `ref` + `onMount`/`onCleanup`, directives (`use:`) |
| **Port** (`Port.inbound/outbound`, `emit`, `stream`, `subscription`) | Schema-validated channels to the host page (`EmbedHandle.ports`) | Props/callbacks across a framework boundary |
| **Submodel** (+ `defineView`, OutMessage, **`reflect*`** helpers) | Child Model/Message/update/view embedded in a parent. `reflect*` (Function.dual) conforms a Submodel to an external source of truth (URL, server push, storage, sibling) **without emitting an OutMessage** | A component + its local signals; controlled vs uncontrolled props |
| **Html builders** (`h.On*`, attributes) | Declarative event bindings in views | JSX event props |
| Foldkit modules: `navigation`, `url`, `route`, `file`, `fieldValidation`, `calendar`, `canvas`, `asyncData`, `http`, `managedResource`, `port` | Built-ins that already cover several Solid primitives | — |
| **foldkit-plus** Mirror (URL/KV), Remote, Sync, Mixins Behaviors, Surface | Secondary representations, server cache, replication, view behaviors, boundaries | storage/url primitives, `db-store`, directives, context |

## 3. The "Foldkit primitive" pattern
A primitive is a module exporting **whichever of these parts it needs**, all typed to its own small Model/Message and designed to be **lifted** into any parent:

```ts
// @pocket/foldkit-primitives/media-query (sketch)
export const Model = Schema.Struct({ matches: Schema.Boolean })             // state slice (replayable)
export const Message = Schema.TaggedUnion({ MediaQueryChanged: { matches: Schema.Boolean } })
export const init = (query: string): Model => ({ matches: false })          // SSR-safe default
export const update = (model, msg) => [{ ...model, matches: msg.matches }, []]
export const subscriptions = (query: string) =>                            // effect + lifecycle
  Subscription.make<Model, Message>()((entry) => ({
    mediaQuery: Subscription.persistent(
      Stream.callback((emit) => { /* matchMedia listener → emit MediaQueryChanged; release removes listener */ })
    ),
  }))
export const reflectMatches = …                                             // optional: external truth
// parent usage:
//   Model: { ..., prefersDark: MediaQuery.Model }
//   update: route MediaQuery messages via Submodel pattern
//   subscriptions: Subscription.lift(MediaQuery.subscriptions("(prefers-color-scheme: dark)"))({ toChildModel: m => m.prefersDark, toParentMessage: m => GotPrefersDark({ message: m }) })
```

**Kinds of primitives** (choose the lightest that fits):
| Kind | Parts | Use when | Examples |
|---|---|---|---|
| **A. Stream source** | Subscription entry only (emits the *parent's* Message via a mapper); no child Model | Parent just needs events | window resize, keyboard, visibility, online/offline, timers, broadcast channel |
| **B. State primitive** | Submodel (Model/Message/update) + Subscriptions/Resources + `reflect*` | State with invariants worth encapsulating | pagination, history/undo, selection, tween/spring, presence, state machine, geolocation watcher |
| **C. Resource primitive** | ManagedResource (+ Messages for acquired/released/data) | Long-lived connections/handles | websocket, SSE, media devices, workers, audio, IndexedDB/SQLite handles |
| **D. Element primitive** | Mount (and a foldkit-plus **Behavior** wrapper) | DOM-element-scoped observation/interaction | resize/intersection/mutation observers, gestures, autofocus, input mask, bounds, scroll, fullscreen on element |
| **E. Command primitive** | `Command.define` Effects | One-shot imperative ops | clipboard write, share, fullscreen request, permission request, script loading, file upload |
| **F. Pure helpers** | Functions over data (no effects) | Derived data | date/i18n formatting, masonry layout math, range, deep compare, immutable ops, keyed list diffs |

**Design rules for all primitives**
1. **Model is the owner.** Primitives never hide state in closures; anything that affects rendering is a Message → Model fact (replay/time-travel safe).
2. **SSR-safe `init`.** No DOM access in `init`; subscriptions/resources/mounts only run in the browser.
3. **Liftable.** Provide `subscriptions`/`resources` as Foldkit records so parents use `Subscription.lift` / `ManagedResource.lift`; Messages are routed via the standard Submodel pattern.
4. **Controlled by default, `reflect*` for external truth.** E.g. `Pagination.reflectPage(fromUrl)`.
5. **Effect-native.** Streams via `Stream.callback`/`acquireRelease`; services (Clock, KeyValueStore, HttpClient) come from Effect so tests use test Layers.
6. **Pause on time-travel** (`viewStateChanges` for Mounts; Subscriptions keyed on a Model flag if needed).
7. **Dual support** (see components design §0.5a): core uses vanilla Foldkit; `/plus` adds Mirror/Behavior/Surface wrappers.

## 4. solid-primitives → Foldkit mapping (all ~80 packages)
Legend: **Built-in** = already in Foldkit/foldkit-plus/Effect; kinds A–F from §3.

### Events, input & sensors
| solid-primitives | Foldkit equivalent | Kind |
|---|---|---|
| event-listener | `Subscription.fromEvent` / `fromEventFilterMap` (**built-in**); `h.On*` in views | A (built-in) |
| keyboard | fromEvent + key-state tracking; hotkeys (`@pocket/foldkit-hotkeys`) | A/B |
| mouse, pointer | fromEvent (window) or Mount (element); pointer position as state | A/D |
| gestures | Mount emitting gesture Messages (pan/pinch/swipe) | D |
| scroll | fromEvent / Mount; scroll position state | A/D |
| active-element | fromEvent `focusin/focusout` on document | A |
| autofocus | Mount (`element.focus()` on insert) | D |
| input-mask | Mount on input + pure mask functions | D + F |
| cursor | Command/Mount setting cursor style | E/D |
| page-visibility | `Subscription.persistent(fromEvent(document, "visibilitychange"))` | A |
| idle | Subscription with timers + activity events | A/B |
| connectivity | persistent fromEvent `online/offline` | A |
| geolocation | ManagedResource (`watchPosition`, requirement = tracking enabled) | C |
| devices | ManagedResource (`mediaDevices` enumerate + `devicechange`) | C |
| media | persistent Subscription on `matchMedia` (see §3 example); breakpoints helper | A/B |
| platform | pure detection helpers (or Effect service) | F |
| permission | Command (query/request) + Subscription on `change` | E/A |

### DOM observation & layout
| solid-primitives | Foldkit equivalent | Kind |
|---|---|---|
| resize-observer | Mount (`ResizeObserver`), window size via persistent Subscription | D/A |
| intersection-observer | Mount (per element) → `EnteredViewport/LeftViewport` | D |
| mutation-observer | Mount | D |
| bounds | Mount measuring rect on resize/scroll | D |
| masonry | pure layout math + measured sizes Messages | F + D |
| virtual | `@pocket/foldkit-virtual` | D + B |
| fullscreen | Command (request/exit) + Subscription (`fullscreenchange`) | E/A |
| styles | pure helpers / Mixins Styles | F |
| raf | `Subscription.animationFrame` (**built-in**, deltaTime) | A (built-in) |
| marker | pure text-highlighting helper → view | F |
| jsx-tokenizer, refs, props, event-props, destructure, controlled-props, keyed, context, lifecycle, rootless, memo, signal-builders, trigger, deep, static-store, mutable, immutable, set, map, destructure | **Not needed:** Solid-reactivity-specific. Foldkit equivalents are the Model, Submodels, views and Schema data types (`HashMap`, `HashSet` in Effect) | — |
| transition-group, presence | Submodel tracking enter/exit phases + Mount for `transitionend`/`animationend` | B + D |

### Timing & animation
| solid-primitives | Foldkit equivalent | Kind |
|---|---|---|
| timer | Subscription (`Stream.tick`/`Schedule.spaced`) keyed on a running flag; or Command `Effect.sleep` | A/E |
| scheduled (debounce/throttle/leading) | Commands with Effect `Stream.debounce`/`throttle` or token-cancelled `Effect.sleep` | E |
| tween, spring | Submodel (value, target, velocity) + `Subscription.animationFrame` while animating | B |
| date | pure helpers over Effect `DateTime` + `Clock` | F |

### Storage, data & network
| solid-primitives | Foldkit equivalent | Kind |
|---|---|---|
| storage, cookies | **foldkit-plus `Mirror.kv`** (+ Effect `KeyValueStore`); cookies via Command | Built-in (plus) / E |
| db-store | **foldkit-plus Remote / Sync** | Built-in (plus) |
| fetch, resource, promise | Commands with Effect `HttpClient`/`HttpApiClient`; Foldkit `http`, `asyncData` (**built-in**) | E (built-in) |
| graphql | Command wrapping a GraphQL client (or HttpClient) | E |
| websocket | ManagedResource (acquire socket; messages via Subscription on the resource stream) or Effect `Socket` | C |
| sse | ManagedResource / Subscription over `EventSource` or Effect `Sse` | C/A |
| stream (media streams) | ManagedResource (camera/mic; d.ts example is exactly a camera) | C |
| broadcast-channel | persistent Subscription + Command to post | A/E |
| workers | ManagedResource (+ Effect `unstable/workers` / Rpc worker protocol) | C |
| upload, filesystem | Foldkit `file` (**built-in**) + Commands (pocket Storage upload with progress) | E |
| clipboard | Command (write/read) + Subscription (paste events) | E/A |
| share | Command (`navigator.share`) | E |
| script-loader | Command (idempotent load) or ManagedResource | E/C |
| audio | ManagedResource (AudioContext/element) + Messages for playback state | C |
| analytics | Command sink / Effect service; or Subscription to app Message log | E |

### App state patterns
| solid-primitives | Foldkit equivalent | Kind |
|---|---|---|
| pagination | Submodel (page, pageSize, total) + `reflectPage` (URL) | B |
| history (undo/redo) | Submodel generic over a child Model (past/present/future); Foldkit DevTools already time-travels for debugging | B |
| selection (text selection) | Subscription `selectionchange` + Command to set | A/E |
| state-machine | Submodel = state machine by construction (tagged Model + update); optional Zag machine adapter | B |
| flux-store, event-bus, event-dispatcher | **Not needed:** Messages + update are the store/bus; cross-tree events via parent Message routing or Ports | — |
| i18n | Submodel (locale) + pure translation helpers; Effect `Context.Reference` for locale | B/F |
| range | pure helpers | F |
| match | Effect `Match` / exhaustive `update` | — |
| share (multiple meanings), utils | pure helpers | F |
| rootless | Not applicable (no owner tree) | — |

## 5. Insights
1. **Most "reactivity plumbing" primitives disappear.** About a quarter of solid-primitives exist to manage Solid's reactive graph (props, refs, memo, trigger, signal-builders, rootless, context, stores). Foldkit's single Model + Messages make them unnecessary.
2. **Foldkit already ships the core effect primitives:** Subscription (`fromEvent`, `animationFrame`, `persistent`, `lift`, `aggregate`), ManagedResource (`make`, `tag`, `lift`), Command, Mount, Port, `http`, `asyncData`, `file`, `navigation`, `url`. The gap is **a curated library of ready-made entries and Submodels** on top.
3. **Submodels are the equivalent for *stateful* primitives** (pagination, tween, history, presence), with `reflect*` covering Solid's controlled/uncontrolled patterns more explicitly.
4. **Subscription/ManagedResource entries are the equivalent for *effectful sources*** (events, observers, sockets, devices), and `lift` gives the same "compose anywhere" ergonomics as calling a primitive inside a component.
5. **Mounts (+ Mixins Behaviors) are the equivalent of directives** (`use:` / ref-based primitives).
6. **Replayability is the extra constraint.** Primitives must record DOM-derived facts as Messages. Solid primitives don't need this, so ports aren't 1:1.

## 6. Proposal: `@pocket/foldkit-primitives`
**Structure** (one subpath per primitive, tree-shakable):
```
@pocket/foldkit-primitives/
  events/        window-size, visibility, online, keyboard-state, pointer, scroll, active-element, idle, clipboard-events
  observers/     resize, intersection, mutation, bounds   (Mounts + /plus Behaviors)
  media/         media-query, breakpoints, prefers (dark/reduced-motion/contrast)
  time/          timer, interval, debounce/throttle command helpers, now (Clock)
  motion/        tween, spring, presence/transition   (Submodels + animationFrame)
  device/        geolocation, media-devices, media-stream, permissions, fullscreen, audio   (ManagedResources + Commands)
  net/           websocket, sse, broadcast-channel, worker   (ManagedResources over Effect Socket/Sse/Worker)
  state/         pagination, history (undo/redo), selection-set, state-machine helpers, i18n locale
  dom/           autofocus, input-mask, script-loader, share, cursor
  plus/          Mirror/Behavior/Surface wrappers (optional peer deps)
```
**Every primitive ships:**
- Model/Message/update (if stateful)
- Subscriptions/ManagedResources/Mounts/Commands as liftable records
- `reflect*` helpers
- SSR-safe `init`
- Test Layers (fake Clock, fake matchMedia/observers)
- A docs page with **side-by-side "Solid primitive → Foldkit"** examples, helping Solid developers adopt Foldkit
- A `/plus` wrapper where relevant

**Build order** (by pocket admin needs):
1. media-query/prefers
2. window-size/resize/intersection
3. visibility/online/idle
4. timer/debounce
5. pagination/history
6. websocket/sse (realtime; may defer to foldkit-remote)
7. clipboard/share/fullscreen
8. tween/spring/presence
9. geolocation/devices

**Upstream opportunity:** several of these (media-query, resize, intersection, timers) are generic enough to propose to Foldkit core or as an official community package.

## 7. Open questions
- Can a Subscription entry be written generically for any parent Model (Kind A mappers), or must it always lift from a child Model? (`Subscription.lift` expects a child subscriptions record.)
- ManagedResource `tag`/`.get` ergonomics for Commands in lifted children.
- Mount args-change semantics (also open in the components plan).
- How Foldkit DevTools displays primitive Messages; naming conventions to keep logs readable (`Media.MediaQueryChanged`).
- Overlap with Foldkit's own UI/components (foldkit.dev) to avoid duplicating existing helpers.
