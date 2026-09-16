# Using the Shadcn Labs Ecosystem in pocket (with Effect & Foldkit)

Drafted 2026-09-16. Builds on [shadcn-labs-ecosystem.md](shadcn-labs-ecosystem.md) (project list), [foldkit-to-react.md](foldkit-to-react.md) (interop options) and [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md) (architecture). Engine details for each project (Takumi, Forme, Satori, Ink/OpenTUI, Eve/Flue, Base UI) are from the project descriptions and are **not yet verified**. Check APIs and runtime support before implementing.

## 1. Principles
1. **Registry items are source, not dependencies.** shadcn-labs blocks are copied into the repo via `npx shadcn add`, so we can edit them, wrap them in Effect, and port them to Foldkit. No runtime lock-in.
2. **Every renderer sits behind an Effect contract.** Email, PDF, OG image, TUI and agent UI rendering are `Context.Service` interfaces with Schema-typed inputs and `TaggedError` failures. shadcn-labs blocks are the *default adapter*, never the contract.
3. **React stays at the edges.** React is used for (a) server-side rendering to strings/bytes (email, PDF, OG), (b) user-facing React apps, (c) the Node TUI. pocket core and the Foldkit admin never import React.
4. **Design tokens are shared.** shadcn's CSS variables (`--background`, `--primary`, `--radius`…) are the single theme source for React blocks, Foldkit views (via foldkit-plus Styles), emails, PDFs and OG images.
5. **Generators wrap the shadcn CLI.** `pocket g …` (Effect `unstable/cli`) runs `shadcn add` through `ChildProcessSpawner`, then writes the Effect/Foldkit glue around the copied source.

## 2. Architecture overview
```
                ┌──────────── pocket.config.ts (theme tokens, collections, plans) ────────────┐
                │                                                                              │
 Effect services (contracts, Schema-typed)                                                     │
   EmailTemplates · Documents(PDF) · OgImages · RichText · Tui · AgentUi · UiRegistry          │
                │                                                                              │
 Adapters (React edge packages)                    Foldkit side                                │
   email-emailcn (jsx-email/React Email)             admin UI (Foldkit + foldkit-plus)         │
   pdf-pdfcn (Takumi/Forme)                          Foldkit ports of registry blocks          │
   og-ogimagecn (Satori)                             <pocket-*> web components (foldkit-to-react §2)
   tui-termcn (Ink/OpenTUI)                          serializeHtml for Foldkit-authored templates
   agent-agentcn (Eve/Flue recipes)                                                            │
   editor-editorcn (Tiptap React) ──► custom element ──► used inside Foldkit admin             │
                │                                                                              │
 Registry: pocket-registry (startercn-based) serves React + Foldkit + web-component variants ◄─┘
```

## 3. Per-project integration

### 3.1 emailcn → `EmailTemplates`
- **Contract**
  ```ts
  class EmailTemplates extends Context.Service<EmailTemplates, {
    render<T extends TemplateId>(id: T, props: TemplateProps[T]): Effect.Effect<RenderedEmail, TemplateError>
  }>()("pocket/EmailTemplates") {}
  // RenderedEmail = { subject, html, text }; TemplateProps derived from each template's Schema
  ```
- **Adapter `email-emailcn`:** emailcn blocks (auth, receipts, bento stats) copied into `emails/`. Each template file exports `Props` (Schema) and a JSX component. The adapter decodes props with Schema, renders with jsx-email (or React Email) inside `Effect.tryPromise`, and runs inline/minify plugins.
- **Effect wiring:** `Mailer.send` jobs (PersistedQueue/effect-mq) call `EmailTemplates.render`; workflows send as durable activities; spans `email.render`/`email.send`; the admin previews with `Arbitrary`-generated props.
- **Foldkit option:** templates written as Foldkit views rendered by `serializeHtml` plus CSS inlining (a second adapter `email-foldkit`) share components with the admin.
- **Generator:** `pocket g email invoice-paid --from emailcn/receipt` → shadcn add + Props Schema + registration + preview story.

### 3.2 pdfcn → `Documents`
- **Contract:** `Documents.render(id, props) → Effect<Uint8Array, DocumentError>`; `Documents.store(id, props, record)` renders to `Storage` and links the file to a record.
- **Adapter `pdf-pdfcn`:** pdfcn blocks (invoice, report, statement) with Schema props; render server-side (engine per pdfcn docs; verify Bun/Workers support). Runs in a job or worker (`unstable/workers`, Rpc `layerProtocolWorker`) to keep CPU off the request path.
- **Integrations:**
  - **PayKit:** invoice/receipt PDFs on payment events.
  - **Collections:** "export as PDF" action per record/view.
  - **Workflows:** generate, email, store as a durable sequence.
  - **Caching:** `PersistedCache` keyed by props hash (`Hash`/`ohash`).

### 3.3 ogimagecn → `OgImages`
- **Contract:** `OgImages.render(template, props) → Effect<{ png: Uint8Array; etag }, OgError>`.
- **Server:** HttpApi endpoint `GET /api/og/:template/:collection/:id` loads the record (rules applied), renders, caches in Storage + HTTP cache (ETag, ocache).
- **Adapter `og-ogimagecn`:** Satori-based blocks (verify WASM/edge support for Workers).
- **Foldkit:** admin page meta editor shows a live preview by calling the endpoint.

### 3.4 editorcn → rich text field type
- **Field:** `Field.richText({ format: "tiptap-json" | "html" | "markdown" })`, stored encoded; Schema codec validates the Tiptap JSON (or sanitizes HTML); server-side render to HTML via Tiptap's HTML generator (verify) or UnJS `md4x` for markdown.
- **User React apps:** use editorcn components directly (registry item `pocket/rich-text-field` wires them to the Records API via `HttpApiClient` or Atom).
- **Foldkit admin:** wrap the editorcn React editor as a **custom element** (`<pocket-rich-text>`), consumed via `CustomElement.define` with Schema-typed `value` property and `change` event (see foldkit-to-react "Reverse direction"). Keeps React out of the Foldkit bundle except for that lazily loaded element.

### 3.5 termcn → `pocket dev` TUI
- **Scope:** interactive dev console: running services, request log, traces (from the local OTel store), jobs/queues, workflow runs, cluster shards, migrations.
- **Integration:**
  - Ink/OpenTUI React components render state.
  - State comes from Effect: a `SubscriptionRef`/`Stream` per panel (e.g. `Jobs.events`, `Tracer` local store) bridged to React with a small `useStream(stream)` hook running on a `ManagedRuntime`.
  - Keyboard actions dispatch Effects (retry job, cancel workflow).
  - `unstable/cli` owns commands and flags; `pocket dev --tui` launches the termcn app.
- **Alternatives:** effect-boxes (Effect-native TUI layout) or motel for traces only. Keep the TUI behind a `DevConsole` service so it can be swapped.
- **Runtime:** Node/Bun only; not in core.

### 3.6 agentcn + mcpcn → AI surfaces
- **Core stays Effect:** `unstable/ai` (`LanguageModel`, `Tool`, `Toolkit`, `Chat`, `McpServer`) with auto-generated tools per collection (CRUD respecting rules), jobs and workflows. foldkit-plus `Agent` exposes admin UI Messages as tools.
- **agentcn (Eve/Flue recipes):** treat as **recipe references and optional runtime adapters**. Port recipes (support bot, data analyst, onboarding agent) to Effect `Toolkit` + workflows; or run Flue/Eve agents that call pocket through the generated MCP server or Rpc client.
- **mcpcn (MCP app UI on Base UI):** UI components for MCP "apps" (rich tool results in ChatGPT/Claude). pocket's `McpServer` returns resources and UI that use mcpcn blocks for record cards, tables and approval forms. Keep them as registry items that consume Schema-typed tool results.

### 3.7 startercn + skills → pocket's own registry and docs
- **`pocket-registry`** built from **startercn**: docs site, landing page, registry JSON, agent-ready.
- **Item variants for each block:**
  - `react`: shadcn component wired to the pocket client (`HttpApiClient`/`AtomHttpApi`/effect-query)
  - `foldkit`: Foldkit view + foldkit-plus Slots/Styles
  - `element`: web component (Foldkit program via embed) for any framework
  - `server`: email/pdf/og template with Schema props
- **Example items:** `pocket/auth-forms`, `pocket/pricing-table` (PayKit plans), `pocket/billing-portal`, `pocket/realtime-list` (Rpc stream), `pocket/record-form` (generated from collection Schema), `pocket/data-table`, `pocket/file-upload` (Storage + thumbs), `pocket/email-verify`, `pocket/invoice-pdf`, `pocket/og-record`.
- **skills:** publish a `pocket` agent skills pack (collections, rules, generators, swapping adapters, Effect v4 idioms), modeled on shadcn-labs/skills and Effect-TS/skills.

### 3.8 Others
- **shadcn-cssinjs (StyleX):** reference for turning shadcn tokens into typed style objects, useful for foldkit-plus `Style.inline` token helpers.
- **framecn / shadercn / slidecn:** out of core scope; possible marketing site, product tours, onboarding videos (framecn + workflows for rendering).

## 4. Theme & token bridge (React ⇄ Foldkit ⇄ email/PDF/OG)
- **Source:** `pocket.config.ts` → `theme: { tokens }` validated by Schema; emitted as shadcn-compatible CSS variables (`globals.css`), a Tailwind v4 theme, and a TS token object.
- **React blocks:** consume CSS variables (standard shadcn).
- **Foldkit:** foldkit-plus `Style.class`/`Style.inline` using the same classes/variables; a `pocketTheme` Style set per Slot capability (Container, Collection…).
- **Email/PDF/OG:** CSS variables don't work in most email clients or Satori, so resolve tokens to literal values at render time (`EmailTemplates` adapter injects resolved tokens).
- **Admin theming = user theming:** one token change updates admin, app, emails, invoices and OG images.

## 5. Effect ⇄ React glue packages
| Package | Purpose |
|---|---|
| `@pocket/react` | `PocketProvider` (ManagedRuntime + client Layers), hooks: `useRecords`, `useRecord`, `useSubscribe` (Rpc stream), `useAuth`, `useBilling`; built on `@effect/atom-react` + `AtomHttpApi`/`AtomRpc` |
| `@pocket/foldkit-react` | Embed Foldkit programs in React (foldkit-to-react Option 1) |
| `@pocket/elements` | Web components for Foldkit programs (Option 2) |
| `@pocket/render-react` | Shared Effect wrapper for server-side React renderers: `renderToString`/bytes with Schema props, spans, timeouts, worker offload |
| `@pocket/cli` generators | `g email / pdf / og / block / element / page`, calling `shadcn add` then generating glue |

## 6. Generator flow (example)
`pocket g block pricing-table`:
1. `ChildProcessSpawner` runs `npx shadcn@latest add <pocket-registry>/pricing-table` (React variant) into the user app.
2. Read PayKit plan definitions from `pocket.config.ts`; generate `plans.ts` Schema and typed `useBilling` wiring.
3. If `--foldkit`: also emit a Foldkit view + Slots and register it in the admin.
4. Print next steps; add a story/preview entry.
All steps are Effects: traced, with dry-run (`--plan`) support.

## 7. Phasing
- **Phase 1:** token bridge; `EmailTemplates` with emailcn + jsx-email; `@pocket/react` hooks; registry skeleton via startercn.
- **Phase 2:** `Documents` (pdfcn) for PayKit invoices; `OgImages`; rich text field (editorcn + custom element in admin); first registry blocks (auth, pricing, record form, data table); skills pack.
- **Phase 3:** `pocket dev` TUI (termcn); agentcn recipe ports + mcpcn UI for MCP tools; Foldkit/element variants for all registry blocks.

## 8. Risks
- **Single maintainer** across shadcn-labs; mitigated because blocks are vendored source behind our contracts.
- **Engine runtime support** (Satori/Takumi/Forme/Ink on Bun/Deno/Workers) is unverified. Keep renderers in adapters with fallbacks (e.g. Foldkit `serializeHtml` + HTML-to-PDF service).
- **React footprint:** confine to edge packages; lazy-load the editor element in admin.
- **Two UI stacks (React + Foldkit):** maintain one token source and ports-first block contracts so variants don't drift; generate variants where possible.
