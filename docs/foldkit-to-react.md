# Foldkit → React (and other hosts): Interop Options

Researched 2026-09-16 against `foldkit@0.160.0` (peer `effect@4.0.0-rc.115`, `@effect/platform-browser@4.0.0-rc.115`) and `doeixd/foldkit-plus`. Related: [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md), [shadcn-labs-ecosystem.md](shadcn-labs-ecosystem.md).

## Goal
pocket's admin and first-party UI are built in **Foldkit** (Elm architecture on Effect). User apps and registries (shadcn, emailcn, etc.) are mostly **React**. We want Foldkit features and view blocks, including **foldkit-plus Mixins** (Slots/Styles/Behaviors), usable as React components, and ideally in any framework.

## Why it can't be a pure build-time compile
Foldkit views are **TypeScript functions**, not templates: `view(model, h) => Html`. Output depends on runtime Model values, closures and Message constructors. A static compiler could only handle trivial cases. Conversion must happen at **runtime** (adapter) or at **render output** (HTML string).

## What Foldkit exposes (verified in package)
| Fact | Where | Implication |
|---|---|---|
| `Html = VNode \| null`, `Child = Html \| string` | `dist/html/index.d.ts` | View output is plain data (tag/`sel`, `data.attrs/props/class/style/on/hook/key`, children, text) |
| Vendored fork of **snabbdom v3.6.3** | `THIRD-PARTY-NOTICES.md`, `dist/snabbdom` | VNode shape ≈ snabbdom's; fork has changes (hooks, replay/unmount tracking). **Internal API, not a public contract** |
| `Runtime.embed` / `run` / `hydrate` / `makeElement` | `dist/runtime/public.d.ts` | Official embedding in foreign pages, React included (README: "run as a widget inside an existing application, React included") |
| `EmbedHandle = { ports, dispose }`; inbound `send(value) → Exit<void, SchemaError>`, outbound `subscribe(listener) → unsubscribe` | `dist/runtime/hostConnector.d.ts` | Typed, Schema-validated host↔app boundary; `dispose` tears down subscriptions, resources, mounts, commands and DOM |
| `CustomElement.define({ tag, properties, events })` (Schema-typed) | `dist/customElement` | Typed bindings for custom elements (consuming them in Foldkit); a program can be exposed as a web component via embed inside a custom element class |
| `serializeHtml(vnode, options) → string` | `dist/experimental/server/serialize.d.ts` | Server rendering to HTML strings (experimental) |
| `createLazy`/`createKeyedLazy`, Mounts, `ManagedResource`, `Subscription`, `Command` | various | Runtime features a naive VNode→React conversion would need to re-implement |

foldkit-plus Mixins: `slots.<name>.attrs()` "resolves the view's own attributes plus every Style/Behavior attached to [the slot] into ordinary Foldkit attributes". **Styles become plain attributes before the VNode exists**; Behaviors may add event handlers or a **Mount** (lifecycle), which needs runtime support.

---

## Option 1: Embed wrapper (works today) ✅ recommended default
React owns a container `<div>`; Foldkit owns everything inside it.

```tsx
// @pocket/foldkit-react (sketch; verify exact embed signature)
export function FoldkitApp<P extends Ports>({ program, flags, inbound, onOutbound }: Props<P>) {
  const ref = useRef<HTMLDivElement>(null)
  const handle = useRef<EmbedHandle<P>>()
  useEffect(() => {
    handle.current = Runtime.embed(program, { container: ref.current!, flags })
    const unsubs = Object.entries(onOutbound ?? {}).map(([name, fn]) =>
      handle.current!.ports[name].subscribe(fn))
    return () => { unsubs.forEach((u) => u()); handle.current?.dispose() }
  }, [program])
  useEffect(() => {                       // props → inbound ports (Schema-validated)
    for (const [name, value] of Object.entries(inbound ?? {})) handle.current?.ports[name].send(value)
  }, [inbound])
  return <div ref={ref} />
}
```
- ✅ Full fidelity: Commands, Subscriptions, Mounts, ManagedResources, Mixins (Styles and Behaviors), foldkit-plus Sync/Agent all work unchanged.
- ✅ Typed boundary: ports are Schema codecs; invalid props are rejected with a `SchemaError` Exit.
- ✅ Clean teardown (`dispose` is idempotent).
- ❌ Black box to React: no React children inside, no React context, one runtime per instance, separate render cycles.
- **Use for:** whole features: admin panels, records table, billing portal (PayKit), auth flows, realtime views, devtools.

## Option 2: Web components (framework-agnostic) ✅
Wrap Option 1 inside a custom element class: `connectedCallback → embed`, `disconnectedCallback → dispose`, attributes/properties → inbound ports, outbound ports → `CustomEvent`s.
- ✅ Works in React 19 (native custom element props/events), Vue, Svelte, Solid, Astro, plain HTML, server templates.
- ✅ Best distribution format for a **framework-agnostic pocket UI registry**.
- ❌ Same black-box trade-offs as Option 1; SSR needs Declarative Shadow DOM or `serializeHtml` + hydrate.
- Generate wrappers from port Schemas: `pocket g element <program>` emits the element class, TS typings (`JSX.IntrinsicElements` augmentation) and a React wrapper.

## Option 3: VNode → React adapter (true conversion) ⚠️ scoped project
React owns state and rendering; Foldkit supplies `Model`, `update` and `view`.

```tsx
// toReact: sketch
export const toReact = <M, Msg>(app: { init: M; update: (m: M, msg: Msg) => [M, Commands]; view: View<M, Msg> }) =>
  function Component(props) {
    const [model, dispatch] = useReducer((m, msg) => runUpdate(app.update, m, msg), app.init)
    return vnodeToReact(app.view(model, makeBuilder(dispatch)), dispatch)
  }

const vnodeToReact = (node: Child, dispatch): ReactNode =>
  typeof node === "string" || node === null ? node
  : node.text !== undefined && !node.sel ? node.text
  : createElement(tagOf(node.sel), {
      key: node.key,
      className: classOf(node.data?.class, node.sel),   // sel "div.foo#bar" parsing
      style: node.data?.style,
      ...node.data?.attrs, ...node.data?.props,         // map "class"/"for"/boolean attrs
      ...mapEvents(node.data?.on, dispatch),            // click → onClick
      ref: mapMountHooks(node.data?.hook),              // insert/destroy → ref callback / effect
    }, node.children?.map((c) => vnodeToReact(c, dispatch)))
```

**Mapping table**
| Foldkit/snabbdom | React | Difficulty |
|---|---|---|
| `sel` (`tag#id.class`) | element type + `id`/`className` | easy |
| `data.attrs`, `data.props` | props (rename `class`, `for`, `tabindex`; controlled `value`/`checked`) | easy–medium (controlled inputs) |
| `data.class` (`{name: bool}`), `data.style` | `className`, `style` | easy |
| `data.on` handlers dispatching Messages | `onX` → `dispatch(msg)` | easy |
| `key` | `key` | easy |
| foldkit-plus **Styles** (resolved attrs) | same as attrs | **free** |
| foldkit-plus **Behaviors**: events | `onX` | easy |
| `hook.insert/destroy`, **Mounts**, Behavior Mounts | ref callbacks + `useEffect` cleanup | medium |
| `createLazy` memoization | `React.memo` / skip (React diffs anyway) | medium |
| **Commands** (Effects returned by update) | Effect runner in the reducer's effect phase (`ManagedRuntime`) | hard |
| **Subscriptions**, `ManagedResource` | `useEffect` with Effect Scope per subscription | hard |
| Submodels / `defineView` | nested components or inline | medium |
| Custom elements, SVG/MathML namespaces | `createElement` with proper tag names | medium |
| Foldkit internal frame/dispatcher/boundary registry | must be bypassed or emulated | **hard, internal** |

- ✅ Real React components: composable, React DevTools, context, SSR with React, shippable as **shadcn registry source**.
- ❌ Re-implements part of Foldkit's runtime; depends on **internal VNode shape** (vendored snabbdom fork), which can break between Foldkit releases.
- **Scope it to "view-only blocks":** pure `view(props)` + Styles + simple event Behaviors, no Mounts, Commands or Subscriptions. That covers most registry UI (cards, tables, pricing, forms display) and avoids the hard rows.
- Ask the Foldkit maintainer for a **public "render to external VDOM" / renderer interface**; that would make this a supported path.

## Option 4: HTML string rendering (static outputs) ✅
`serializeHtml(view(model))` → HTML string (experimental API).
- ✅ No React. Use for **emails** (alternative to jsx-email when templates are Foldkit views), **PDF** (HTML-to-PDF engines), **OG images** (HTML/CSS to Satori-like engines, verify), SSR islands then `hydrate`.
- ❌ Static only; interactivity requires `hydrate` with Foldkit on the client.
- Styles from Mixins serialize as attributes; email needs CSS inlining (reuse jsx-email's inline plugin or juice).

## Reverse direction (React inside Foldkit)
Occasionally needed for editorcn (Tiptap React), emailcn previews, etc.:
- Wrap React components as **custom elements** (e.g. `@r2wc/react-to-web-component`, verify) and consume via `CustomElement.define` with Schema-typed props/events.
- Or mount via a Foldkit **Mount** (`createRoot` on insert, `unmount` on destroy).

---

## Comparison
| | Fidelity | React-native feel | Framework-agnostic | Effort | Stability risk |
|---|---|---|---|---|---|
| 1. Embed wrapper | Full | Low (black box) | React only (per wrapper) | Low | Low (public API) |
| 2. Web components | Full | Medium | **Yes** | Low–medium | Low |
| 3. VNode → React | View-only subset | **High** | No | High | **High** (internals) |
| 4. HTML string | Static | n/a | Yes | Low | Medium (experimental) |

## Recommendation for pocket
1. **Ship `@pocket/foldkit-react`** (Option 1) in Phase 1: `<FoldkitApp>` + typed hooks (`usePort`) generated from port Schemas.
2. **Ship `@pocket/elements`** (Option 2) for the framework-agnostic registry; generator emits element + React/Vue typings.
3. **Use Option 4** inside `EmailTemplates`/`Documents`/`OgImages` adapters as a Foldkit-native renderer alternative to jsx-email/pdfcn/ogimagecn.
4. **Prototype Option 3** as `@pocket/foldkit-react-views`, limited to view-only blocks + foldkit-plus Styles; gate behind "experimental" until Foldkit offers a public renderer hook.
5. Design pocket UI blocks as **ports-first contracts** (Schema inbound/outbound) so the same block works through Options 1, 2 and 3.

## Next steps
- Confirm exact `Runtime.embed` signature and program/ports definition from foldkit.dev/core/embedding.
- Spike: embed wrapper + web component for a foldkit-plus Surface view with a Style attached.
- Spike: `vnodeToReact` on 3 view-only components; measure what breaks (controlled inputs, SVG, Behaviors).
- Open a discussion with the Foldkit maintainer about a public VDOM/renderer interface and foldkit-plus Mixins in non-Foldkit hosts.
