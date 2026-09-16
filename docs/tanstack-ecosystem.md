# TanStack Ecosystem (researched 2026-09-16)

Source: GitHub API for the TanStack org (stars, last push) and npm (latest versions). TanStack libraries are mostly **headless, framework-agnostic cores** (`*-core`) with thin adapters (React, Solid, Vue, Svelte, Angular, Lit, Preact). **No TanStack package depends on Effect-TS** (TanStack DB has its own unrelated `createEffect` API). Related docs: [effect-ecosystem.md](effect-ecosystem.md), [react-in-foldkit.md](react-in-foldkit.md), [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md).

## Inventory

### Data & state
| Project | ★ | Packages (latest) | What |
|---|---|---|---|
| **Query** | 50.3k | `@tanstack/query-core` 5.103.0; react/solid/vue/svelte (6.2.0)/lit/angular adapters; `*-query-devtools`, `*-persist-client`, `query-sync/async-storage-persister`, `query-broadcast-client-experimental`, `eslint-plugin-query` | Async server-state: caching, dedupe, retries, invalidation, infinite queries, mutations, SSR hydration |
| **DB** | 3.9k | `@tanstack/db` 0.9.2, `db-ivm` (incremental view maintenance), `react-db` 0.4.1, `solid-db`, `vue-db`, `svelte-db`, `angular-db`; collections: `query-db-collection`, `electric-db-collection`, `powersync-db-collection`, `rxdb-db-collection`, `trailbase-db-collection`; persistence: `db-sqlite-persistence-core` + browser/node/expo/react-native/capacitor/tauri/electron SQLite; `offline-transactions` 1.0.56; `react-router-with-db` | "Reactive client store for your API": typed collections, live queries (differential dataflow, sub-ms), optimistic mutations with rollback, sync-engine backends, offline |
| **Store** | 894 | `@tanstack/store` 0.11.1 | Framework-agnostic reactive store (powers Form/Router internals) with adapters |
| **Persist** | 33 | (not yet on npm under that name) | Utilities/hooks to persist stores, state and collections |
| **Pacer** | 768 | `@tanstack/pacer` 0.22.0, `pacer-lite` 0.2.2 | Debounce, throttle, rate limit, queue, batch (sync + async) |

### Routing & full-stack
| Project | ★ | Packages | What |
|---|---|---|---|
| **Router** | 15.1k | `router-core` 1.171.30, `react-router` 1.170.36, `solid-router`, `vue-router` 1.170.33, `router-plugin` (file routes, code-splitting), `eslint-plugin-router` | Fully type-safe router: typed search params (Standard Schema validators), loaders, caching, preloading |
| **Start** | (in router repo) | `react-start` 1.168.54, `solid-start` 1.168.52, `react-start-rsc` 0.1.53 (React Server Components), `start-*-core`, `start-storage-context`, `create-start`, `eslint-plugin-start` | Full-stack framework on Router + Vite: server functions, SSR/streaming, server routes, middleware, deploy anywhere (Nitro-based presets) |

### UI primitives (headless)
| Project | ★ | Packages | What |
|---|---|---|---|
| **Table** | 28.4k | `table-core` 9.2.4, react/solid/vue/svelte/angular/lit adapters, `table-devtools` 9.2.0 | Headless datagrid: sorting, filtering, grouping, pagination, column sizing/pinning/visibility, row selection, expansion |
| **Virtual** | 7.1k | `virtual-core` 3.17.11 + adapters | Virtualized lists/grids (fixed/dynamic sizes, infinite, window scroll) |
| **Form** | 6.7k | `form-core` 1.33.5, `react-form` 1.33.5, vue/angular/solid/lit/svelte adapters, `form-devtools`, adapters for zod/valibot/yup (0.42.1; Standard Schema supported natively) | Headless, type-safe forms: field-level validation (sync/async, debounced), arrays, linked fields, SSR |
| **Select** | 278 | (not yet on npm) | Select / multi-select / autocomplete primitives |
| **Ranger** | 837 | `@tanstack/ranger` 0.0.4 | Range / multi-range slider utilities |
| **Time** | 622 | (not yet on npm) | Headless time and calendar component utilities |
| **Hotkeys** | 715 | `hotkeys` 0.8.0 + react/solid/vue/svelte/angular/lit/preact adapters, devtools | Type-safe keyboard shortcuts |
| **Charts** | 741 | `charts` 0.18.0, `charts-scales` 0.18.0 | Tiny visualization grammar: responsive, accessible, **server-rendered** charts on granular D3 primitives |

### AI & durable execution
| Project | ★ | Packages | What |
|---|---|---|---|
| **AI** | 3.1k | `@tanstack/ai` 0.54.0, `ai-client` 0.31.1; providers: openai, anthropic, gemini, mistral, groq, grok, cohere, bedrock, ollama, openrouter, perplexity, byteplus, fal, elevenlabs, vercel-gateway; agent bridges: `ai-claude-code`, `ai-codex`, `ai-opencode`, `ai-acp` (Agent Client Protocol); `ai-mcp`; **code mode** (`ai-code-mode`, `-skills`) with isolates (`ai-isolate-node/quickjs/cloudflare`), `ai-sandbox`; `ai-persistence`, `ai-compaction`; framework adapters (react/solid/vue/svelte/angular/preact); devtools | Provider-agnostic, type-safe AI SDK: streaming chat, tool calling, agents, multimodal |
| **Workflow** | 205 | `workflow-core` 0.0.4, `workflow-runtime` 0.0.3, `workflow-store-drizzle-postgres` 0.0.5 | Type-safe durable execution for agents/workflows: resumable runs, append-only history, compensable steps (very early) |

### Tooling & content
| Project | ★ | Packages | What |
|---|---|---|---|
| **DevTools** | 497 | `devtools` 0.14.2, `devtools-ui`, `devtools-event-bus`/`-event-client`, `devtools-vite`, `devtools-a11y`, react/solid/vue/preact wrappers | Framework-agnostic devtools panel with **custom plugins** |
| **CLI** | 1.3k | `@tanstack/cli` 0.71.0, `@tanstack/create` 0.70.0 | Scaffolding, add-ons, **MCP server**, agent skills installation |
| **Intent** | 331 | `@tanstack/intent` 0.4.0 | CLI for library maintainers to generate, validate and ship **Agent Skills** with npm packages |
| **Markdown** | 394 | `markdown` 0.0.15 | Tiny fast Markdown parse/render |
| **Highlight** | 77 | `highlight` 0.1.0 | Tiny synchronous syntax highlighting |
| **Redact** | 254 | `redact` (npm name TBD; experimental) | Alternative React implementation, API-compatible, smaller/faster |
| Config | 392 | `eslint-config`, `vite-config`, `publish-config`, `typedoc-config` | Library maintenance tooling |

---

## Overlap with Effect (choose deliberately)
| Concern | TanStack | Effect | Guidance for pocket |
|---|---|---|---|
| Server state / caching | Query | `HttpApiClient` + `unstable/reactivity` Atom (`AtomHttpApi`, `AtomRpc`) | Effect-native default; offer **effect-query** (voidhash, v4) bridge for Query users |
| Client DB / live queries / optimistic | **DB** | Atom + `Reactivity` invalidation; `unstable/eventlog` (local-first) | **TanStack DB is stronger** for live queries and optimistic writes; build a pocket collection (see below) |
| Rate limit / debounce / queue | Pacer | `Schedule`, `RateLimiter`, `Queue`, `Stream.debounce/throttle` | Effect on server; Pacer fine for UI-only |
| Durable workflows | Workflow (0.0.x) | `unstable/workflow` + cluster | Effect (far more mature) |
| AI | AI (many providers, code mode, agent bridges) | `unstable/ai` + `@effect/ai-*` (4 providers) | Effect as core contract; TanStack AI as a provider **adapter** behind `LanguageModel` for breadth; `ai-code-mode` isolates are unique |
| Validation | Standard Schema consumers | Schema → `toStandardSchemaV1` | Works as-is (Form, Router search params, Start server fns) |
| Stores | Store | `SubscriptionRef`, Atom | Effect in app code |
| Devtools | DevTools plugins | Effect DevTools, OTel | Ship a **pocket TanStack DevTools plugin** for React users |

## How pocket can use TanStack
1. **TanStack Start as a first-class host.** Mount pocket via `HttpRouter.toWebHandler` in Start server routes; call Effect services from server functions through a `ManagedRuntime`; Router search params validated with Effect Schema (Standard Schema). Template: `pocket new --template start` (via `@tanstack/cli` add-on or our CLI).
2. **`@pocket/tanstack-db` collection.** A custom DB collection syncing from pocket's realtime (Rpc stream or SSE) with mutations mapped to the Records API. Rules are enforced server-side, and optimistic rollback uses typed errors. Offline via `offline-transactions` + SQLite persistence. Highest-value React/Solid/Vue client integration; parity with electric/powersync collections.
3. **`@pocket/tanstack-query`** helpers: query/mutation options generated from the `HttpApi` contract (or recommend effect-query).
4. **Forms from collection Schemas.** Generate TanStack Form configs (Standard Schema validators from Effect Schema, async validators calling rules/unique checks) for the React registry `record-form` block. SCHEMA.md already shows the TanStack Form integration.
5. **Admin & Foldkit: use the cores natively** (framework-agnostic, fits react-in-foldkit.md path 3):
   - `table-core` for the records data grid: Foldkit Model holds table state; view renders rows.
   - `virtual-core` for large record lists and logs.
   - `form-core` optionally for complex admin forms (or Foldkit `fieldValidation`).
   - `hotkeys` (core) for admin keyboard shortcuts; `charts` for server-rendered dashboard charts (also usable in emails/PDFs as SVG, verify).
   - `ranger` for range filters.
6. **AI breadth:** `LanguageModel` adapter over `@tanstack/ai` for providers Effect lacks (Gemini, Mistral, Groq, Ollama, Bedrock…); expose pocket MCP tools to `ai-mcp`; consider `ai-code-mode` + isolates for "agent writes a hook/query" features (sandboxed).
7. **Agent skills distribution:** use **TanStack Intent** (or the same convention) to ship pocket's skills inside npm packages; `@tanstack/cli` MCP server as a reference for `pocket mcp`.
8. **DevTools:** a pocket plugin for TanStack DevTools (requests, realtime subscriptions, auth session, traces) for React/Solid users.

## Community Effect ⇄ TanStack bridges (from effect-ecosystem.md)
- **effect-query** (voidhashcom/effect-query, v4): Query options from Effect Rpc/HttpApi clients.
- **tanstack-db-atom** (v3, stale): DB collections as Effect Atoms. A v4 rewrite is a gap pocket could fill.
- **tiesen243/effect-tanstack-query**: Query bindings (check version).
- alchemy examples `tanstack-rpc-drizzle`: Start + Effect Rpc + Drizzle.

## Caveats
- Several projects are early (`workflow` 0.0.x, `ranger` 0.0.4, `markdown` 0.0.x, `select`/`time`/`persist` not yet published under those names); DB is pre-1.0 (0.9.x).
- Versions move very fast (Router/Start ship near-daily). Pin and use `@tanstack/cli`/renovate.
- Adapter coverage differs per library (e.g. Svelte Query is on 6.x while React Query is 5.x).
