# React → Foldkit: Using the React Ecosystem from Foldkit

Drafted 2026-09-16 against `foldkit@0.160.0`. Companion to [foldkit-to-react.md](foldkit-to-react.md) (the opposite direction) and [shadcn-labs-integration.md](shadcn-labs-integration.md).

## Short answer
- **General "compile React → Foldkit": no.** React components are programs, not templates: hooks (`useState`, `useEffect`, context, refs), the reconciler's scheduling, portals and Suspense have no static equivalent in Foldkit's Model/Message/update. A compiler would have to embed React's runtime, which is option 1 anyway.
- **But there are five practical paths.** Ordered by how much of the React ecosystem each unlocks:

| # | Approach | Unlocks | Fidelity | Effort |
|---|---|---|---|---|
| 1 | **React islands via Foldkit `Mount`** | Any React component (Radix/shadcn, Tiptap, charts, maps, editors) | Full (real React) | Low |
| 2 | **React → Web Components**, consumed with `CustomElement.define` | Any React component; also reusable outside Foldkit | Full | Low |
| 3 | **Framework-agnostic cores** (Zag, TanStack *core*, Floating UI, Tiptap core…) | Headless logic behind most React libs, used natively | Native Foldkit | Medium |
| 4 | **JSX/markup transform for presentational components** | shadcn/Tailwind *markup* (blocks, layouts, static cards) | Native Foldkit | Medium (tooling) |
| 5 | **Server-render React to HTML** | Static content (emails, docs, marketing sections) | Static only | Low |
| + | **Agent-assisted porting** | One-off ports of stateful components to real Foldkit | Native | Per component |

---

## 1. React islands via `Mount` (recommended first)
Foldkit's `Mount` is built for this (verified in `dist/mount/index.d.ts`). A Mount is a named per-element side effect whose runtime shape is `f(element, viewStateChanges) => Stream<Message>`. The Stream's scope is **tied to the element's lifetime**: on unmount the runtime interrupts the fiber and runs `acquireRelease` finalizers. `Mount.define` (one-shot → Message) and `Mount.defineStream` (continuous events) build it; args are Schema-typed.

That maps onto a React root:
- **Acquire:** `createRoot(element)` and `root.render(<Component {...args} onChange={emit} />)`
- **Events:** React callbacks push into the Stream (`Stream.callback`), which become Foldkit Messages handled by `update`
- **Release:** `root.unmount()` in the finalizer

```ts
// sketch: verify Mount.defineStream's exact signature and arg handling
import { Effect, Schema, Stream } from "effect"
import { Mount } from "foldkit"
import { createRoot } from "react-dom/client"
import { createElement } from "react"
import { DatePicker } from "@/components/ui/date-picker" // shadcn (Radix) component

export const DatePickerIsland = Mount.defineStream("DatePickerIsland", { value: Schema.String }, (args) =>
  (element) =>
    Stream.callback<Message>((emit) =>
      Effect.acquireRelease(
        Effect.sync(() => {
          const root = createRoot(element)
          root.render(createElement(DatePicker, {
            value: args.value,
            onChange: (v: string) => emit.single(Message.ChangedDate({ value: v })),
          }))
          return root
        }),
        (root) => Effect.sync(() => root.unmount())
      )
    )
)

// in a view:
h.div([h.OnMount(DatePickerIsland({ value: model.date }))], [])
```

**Details to settle:**
- **Prop updates:** when `model.date` changes, does Foldkit re-run the Mount (args differ), or should the island keep the root and re-render? Prefer a single long-lived root with an inbound Stream/`SubscriptionRef` of props → `root.render(...)` on each change, to preserve React state (focus, popovers). Verify Foldkit's Mount identity/args-diff semantics.
- **DOM ownership:** the island's element must have **no Foldkit children** (React owns its subtree). Give it a stable `key`.
- **Controlled state:** keep the source of truth in the Foldkit Model; React component is controlled (`value` + `onChange` → Message). Uncontrolled internal UI state (open/closed popover) can stay in React.
- **Portals** (Radix dialogs/popovers render into `document.body`): work, but z-index/focus traps cross Foldkit DOM. Scope with a portal container element if needed.
- **Styling:** Tailwind/shadcn CSS variables are global, so they apply inside islands too (see token bridge in shadcn-labs-integration.md).
- **Bundle:** load React lazily per island (`import()` inside acquire) so pages without islands don't ship React.
- **Time-travel/DevTools:** historical renders use no-op dispatch per Foldkit docs, so islands must tolerate being re-mounted during replay.

**Package idea:** `@pocket/foldkit-react-island` with `ReactIsland.make(Component, { props: Schema, events: { onChange: Schema } })` → typed Mount + Message constructors. Generated from a component's props Schema.

## 2. React → Web Components → `CustomElement.define`
- Wrap the React component as a custom element (e.g. `@r2wc/react-to-web-component`; verify maintenance/React 19 support, or hand-roll ~40 lines: `connectedCallback → createRoot.render`, attribute/property setters re-render, callbacks → `dispatchEvent(new CustomEvent(...))`).
- Consume in Foldkit with **`CustomElement.define({ tag, properties, events })`**: properties and event `detail` are **Schema-validated**, and the typed builder gives `Value(...)` / `OnChange(...)` factories (verified in `dist/customElement`).
- ✅ Framework-agnostic artifact (works in Foldkit, plain HTML, Vue, Svelte); clean typed boundary; no Foldkit-specific glue per component.
- ❌ Shadow DOM vs Tailwind: use light DOM (no shadow root) so global shadcn styles apply; attribute serialization for complex props → use properties.
- **Best for:** reusable widgets pocket ships in its registry `element` variant, and the editorcn rich text editor in the admin.

## 3. Use the framework-agnostic cores behind React libraries (native Foldkit)
Much of the React ecosystem is a thin React adapter over a vanilla core. Elm-architecture Foldkit pairs naturally with **state machines and pure cores**:

| Need | React lib | Framework-agnostic core to use from Foldkit (verify each) |
|---|---|---|
| Accessible UI primitives (dialog, menu, combobox, tabs, date picker…) | Radix / Base UI / Ark UI | **Zag.js** state machines (`@zag-js/*`): pure machines + `normalizeProps`, used by Ark UI for React/Vue/Solid/Svelte. A Foldkit connector would map machine state to Model and events to Messages |
| Positioning (popovers, tooltips) | Floating UI React | `@floating-ui/dom` in a Mount |
| Tables / virtualization / forms | TanStack Table/Virtual/Form (React adapters) | `@tanstack/table-core`, `@tanstack/virtual-core`, `@tanstack/form-core` (Standard Schema → Effect Schema) |
| Rich text | editorcn / Tiptap React | `@tiptap/core` in a Mount (vanilla editor) |
| Charts | Recharts | ECharts, Chart.js, uPlot, Observable Plot (vanilla) in a Mount |
| Maps | react-map-gl | maplibre-gl in a Mount |
| Drag & drop | dnd-kit | `@dnd-kit/dom` (framework-agnostic rewrite; verify), Pragmatic drag and drop, SortableJS |
| Animation | Motion for React | `motion` (vanilla `animate`) in Mount |
| Command palette / toasts | cmdk, sonner | Zag machines or port |
| Icons | lucide-react | `lucide` (vanilla SVG data) → Foldkit SVG builder |
| Data fetching/cache | TanStack Query | Not needed: Effect `HttpApiClient` + Foldkit Commands (or `@tanstack/query-core`) |

Note Foldkit already ships UI pieces (`calendar`, `fieldValidation`, `file`, `navigation`, `canvas`) and Foldkit UI components (foldkit.dev); check before importing.

**pocket plan:** a `foldkit-zag` connector (generic: machine service → Model slice + Messages + props spreader into `h` attributes) would unlock the entire Ark UI component set natively, styled with shadcn tokens. That's the highest-leverage native path.

## 4. Transform presentational JSX → Foldkit views
For shadcn **markup** (Tailwind classes, layout, static structure, no hooks), a build-time or codegen transform is feasible:
- **JSX factory approach:** compile `.tsx` with a custom `jsxImportSource` (`@pocket/foldkit-jsx`) whose `jsx(type, props, ...children)` produces Foldkit VNodes via `h` (`className` → `class`, `onClick={msg}` → `h.OnClick(msg)`). Lets authors write JSX that *is* Foldkit (no React at runtime). **Verify** Foldkit's h builder can be targeted like this (it's typed per Message; a JSX factory may lose some Message typing).
- **Codemod approach:** `ts-morph`/Babel codemod converts shadcn component files into Foldkit `defineView` functions; `cn()`/`cva` variants → foldkit-plus `Style` variants; `React.forwardRef`/`Slot` (asChild) → Slots; flags any hook usage for manual/agent port.
- Works for: shadcn **blocks** (dashboards, pricing sections, cards, headers), `cva` variant styling (Button, Badge, Card). Not for: Radix-driven interactive components; use path 1/2/3.
- **Registry pipeline:** `pocket g block X --foldkit` runs `shadcn add` → codemod → Foldkit view + Slots; unresolved hooks produce TODOs with a suggested island (path 1) or Zag machine (path 3).

## 5. Server-render React to static HTML
`renderToStaticMarkup` (or jsx-email/React Email renderers) → HTML string → insert into a Foldkit view as native inner HTML (Foldkit has internal `readNativeInnerHtml`/`writeNativeInnerHtml` helpers; check the public API for raw HTML insertion and sanitize). Good for docs content, marketing sections, email previews in admin. No interactivity.

## + Agent-assisted porting
Foldkit's explicit Model/Message/update is very agent-friendly ("agents fall into the pit of success"). For a stateful component worth owning natively: give an agent the React source + a Foldkit skill, and require an Elm-style port with tests (`foldkit/test`, `Scene` helpers). Good for a handful of core admin components (data table, combobox, command palette). Keep the ported set small and tested.

---

## Recommendation for pocket
1. **Now:** `@pocket/foldkit-react-island` (path 1): typed Mount wrapper with long-lived root + props stream, lazy React loading. Unblocks any React component in the admin immediately (editorcn, charts, date pickers).
2. **Registry `element` variant** (path 2) for shareable widgets; consume via `CustomElement.define`.
3. **Invest in `foldkit-zag`** (path 3) for native, accessible primitives styled with shadcn tokens: the long-term replacement for most islands.
4. **Codemod for shadcn blocks** (path 4) inside `pocket g block --foldkit`, falling back to islands/Zag for interactive parts.
5. Use path 5 for static content; agent-porting for a few core components.

## Open questions / verify
- Mount args-change semantics (re-mount vs update) and how to pass a props Stream into a Mount.
- Whether Foldkit exposes a public raw-HTML insertion API.
- Zag.js API fit with Foldkit's attribute builder (event handler shapes, `normalizeProps`).
- JSX factory typing against `HtmlBuilder<Message>`.
- Coordinate with the Foldkit maintainer: an official `foldkit-react-island` or Zag connector may be welcome upstream.
