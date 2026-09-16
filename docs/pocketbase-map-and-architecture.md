# App-in-a-Box on Effect: PocketBase Feature Map & Architecture Approach

Drafted 2026-09-16. Inputs: [effect-v4-api-scope.md](effect-v4-api-scope.md), [effect-schema.md](effect-schema.md), [effect-ecosystem.md](effect-ecosystem.md), [effect-adjacent-projects.md](effect-adjacent-projects.md), [unjs-ecosystem.md](unjs-ecosystem.md). PocketBase reference: pocketbase/pocketbase v0.40.4 (61k★). Feature list is from its docs as I know them; re-check details before copying behavior.

Working name below: **`pocket`** (after this folder); rename freely.

---

## 1. PocketBase feature map

Legend: ✅ covered by an existing library · 🟡 building blocks exist, we must compose/extend · 🔴 must build ourselves

| # | PocketBase feature | Effect-native provider | Other provider (UnJS / 3rd party) | Status | What we build |
|---|---|---|---|---|---|
| 1 | **Single binary / embedded server** | `@effect/platform-node`/`-bun`, `HttpServer`, `HttpRouter`, `Layer.launch`, `NodeRuntime.runMain` | Nitro/h3/srvx (portable deploy targets); Bun `--compile` for a single file | 🟡 | `pocket serve` CLI entry; a single-binary build via `bun build --compile` |
| 2 | **Collections** (base / auth / view types, field types, system fields id/created/updated) | `Schema` (`Class`, `TaggedStruct`), `unstable/schema` `Model.Class` (select/insert/update/json variants, `GeneratedByDb`), `SchemaRepresentation` (**schemas persisted as JSON and rebuilt at runtime**) | Drizzle `effect-schema` (table ↔ schema) | 🟡 | A **collection definition DSL** that compiles to DB table, Schema, API routes, rules, admin UI and client types. Dynamic (runtime-editable) collections via `SchemaRepresentation` |
| 3 | **Embedded SQLite DB** (+ view collections) | `@effect/sql-sqlite-node`/`-bun`/`-wasm`/`-do`, `@effect/sql-pg`, `@effect/sql-d1`, libsql; `SqlClient`, `SqlResolver`, `SqlSchema`, `Statement` | Drizzle `effect-*` subpaths (typed query builder); UnJS db0 | ✅/🟡 | A `Database` service over `SqlClient`; SQLite default, Postgres/D1/libSQL as swaps. Choose Drizzle as the query layer (see §4) |
| 4 | **Migrations** (auto-generated from admin schema changes, JS migrations, up/down) | `unstable/sql` `Migrator` | drizzle-kit (diffing) | 🟡 | Diff collection definitions → migration files; `pocket migrate` CLI (up/down/status/create) |
| 5 | **REST CRUD API** (list/view/create/update/delete, pagination, sort, `filter`, `expand`, `fields`, batch) | `unstable/httpapi` (`HttpApi`, `HttpApiGroup`, `HttpApiEndpoint`, `HttpApiBuilder`), generated OpenAPI/Scalar/Swagger, `HttpApiClient`; `unstable/rpc` as an alternative transport | — | 🟡 | **Filter-expression language** (PB syntax `a = "x" && b ~ "y"`) parser → SQL; expand/relations resolver (batch with `RequestResolver`); generic record endpoints generated per collection |
| 6 | **API rules** (per-collection list/view/create/update/delete rules using `@request.auth.*`, `@collection.*`) | `HttpApiMiddleware`, `HttpApiSecurity`, `RpcMiddleware`, Schema filters | better-auth access control plugin (roles only) | 🔴 | Rule compiler: same expression language → SQL WHERE (list/view) + pre-write checks. **This is PB's secret sauce; build it carefully** |
| 7 | **Realtime subscriptions** (SSE, subscribe to a collection or record, rule-filtered) | `PubSub`, `Stream`, `unstable/encoding` `Sse`, HttpApi streaming responses, `Socket`/`SocketServer`, `unstable/reactivity` `Reactivity` (invalidation keys) | crossws (UnJS WebSockets anywhere); celld/Durable Objects for fan-out on edge | 🟡 | `Realtime` service: publish on record change → rule-filter per subscriber → SSE/WS. Multi-node: swap the PubSub Layer for Redis/pg LISTEN (`@effect/sql-pg` has a dedicated LISTEN connection) |
| 8 | **Auth**: email/password, OAuth2 (many providers), OTP, MFA, impersonation, verification / password reset emails, auth tokens | — (no official Effect auth) | **better-auth** (email/pw, OAuth, magic link, OTP, 2FA/passkeys, organizations, admin/impersonation, API keys, JWT) | ✅ via wrapper | `Auth` service interface; `AuthBetterAuth` Layer (wrap `auth.api.*` in `Effect.tryPromise`, mount handler into HttpRouter, share DB via Drizzle adapter). Map the "auth collection" concept onto better-auth user tables |
| 9 | **Superusers / admin accounts** | — | better-auth admin plugin | 🟡 | Superuser role + bootstrap CLI (`pocket superuser create`) |
| 10 | **File storage** (local or S3, per-record file fields, protected files with tokens) | `FileSystem`, `Path`, `KeyValueStore`, `HttpStaticServer`, `Multipart` | unstorage (fs/S3/R2/…) for blobs; distilled R2/S3 SDKs; Alchemy for bucket provisioning | 🟡 | `Storage` service (put/get/delete/signedUrl/stream); Layers: local fs, S3/R2. File field type + multipart upload endpoints + protected-file tokens |
| 11 | **Image thumbs** (`?thumb=100x100`) | — | UnJS **ipx** (image optimizer), image-meta | ✅ via wrapper | `Thumbnails` service → ipx Layer; cache results in Storage |
| 12 | **Email** (SMTP, templates) | — | **unemail** (18 providers incl. SMTP/Resend/SES) | ✅ via wrapper | `Mailer` service + unemail Layer + in-memory test Layer; templates |
| 13 | **Cron jobs** | `Schedule`, `Cron`, `Effect.repeat`, `unstable/cluster` `ClusterCron` | — | ✅ | `Jobs.cron(name, expr, effect)` registry + admin listing |
| 14 | **Background jobs / queues** (not native to PB) | `unstable/persistence` `PersistedQueue`, `unstable/workflow` (durable), cluster | effect-mq (Postgres/Redis/memory, v4) | ✅ | `Queue` service; PersistedQueue default, effect-mq as swap |
| 15 | **Hooks / extending with code** (`onRecordCreate`, request hooks, custom routes, JSVM/Go) | `Layer`, `PubSub`, `Effect` middleware, `HttpRouter` | UnJS hookable | 🟡 | **Typed hook bus**: `Hooks.on("records.beforeCreate", collection, effect)` with Schema-typed payloads; custom routes = user HttpApi groups merged into the app API. Extensions = Layers (the plugin model, see §3) |
| 16 | **Admin dashboard** (collections editor, records browser, logs, settings, backups) | `unstable/reactivity` Atom + `AtomHttpApi` | **Foldkit** (Elm arch, Schema model) + **foldkit-plus** (Surfaces, Sync, Mirror, Mixins, Agent/MCP) | 🔴 | Admin SPA in Foldkit; generated from collection metadata; foldkit-plus Mixins for theming/slots; foldkit-plus Agent → "admin via MCP" for free |
| 17 | **JS SDK** (auth store, realtime, typed records) | `HttpApiClient` (derived from the API definition), `RpcClient`, `unstable/reactivity` (`AtomHttpApi`, `AtomRpc`), `@effect/atom-react/solid/vue` | effect-query (TanStack Query); ofetch for a non-Effect client | 🟡 | Two SDKs: **Effect SDK** (derived `HttpApiClient` plus realtime Stream) and a **Promise SDK** wrapper for non-Effect users; type codegen for dynamic collections via `SchemaRepresentation.toCodeDocument` |
| 18 | **Settings** (app name, URL, SMTP, S3, rate limits, stored in DB, editable) | `Config`, `ConfigProvider`, `Context.Reference`, `LayerRef` (**swap Layers at runtime**) | UnJS c12 (file config), std-env | 🟡 | Settings = Schema-typed doc in DB + `ConfigProvider` merge (env > file > DB); `LayerRef` to hot-swap Mailer/Storage when settings change |
| 19 | **Logs** (request logs in DB, viewer) | `Logger`, `Tracer`, `Metric`, `unstable/observability` Otlp/Prometheus | @kitlangton/motel (local OTel TUI) | 🟡 | A `Logger` Layer that writes to a logs table + admin viewer; OTLP export optional |
| 20 | **Backups** (zip data dir, S3 upload, scheduled) | `FileSystem`, `Schedule` | UnJS nanotar; unstorage | 🟡 | `Backups` service: SQLite `VACUUM INTO` + tar + Storage upload, cron |
| 21 | **Rate limiting** | `unstable/persistence` `RateLimiter` (+ `adaptive`), `HttpClient.withRateLimiter` | — | ✅ | HttpApi middleware using RateLimiter, per-IP/user/rule |
| 22 | **Batch API / transactions** | `SqlClient.withTransaction`, Drizzle `transaction` | — | 🟡 | `/api/batch` endpoint executing N record ops in one transaction |
| 23 | **CLI** (`serve`, `migrate`, `superuser`, `update`) | `unstable/cli` (`Command`, `Flag`, `Argument`, `Prompt`, completions) | citty/consola (not needed) | ✅ | `pocket` CLI |
| 24 | **Type safety for server extensions** | Everything is Schema/Effect | — | ✅ | — |

### Beyond PocketBase ("additional stuff")
| Feature | Provider |
|---|---|
| **Payments / billing / entitlements** | PayKit (`paykitjs`, Stripe) behind a `Billing` service; `@paykit-sdk/core` (Stripe/Polar/PayPal) as an alternative Layer |
| **Deploy / IaC** (Cloudflare, AWS, Fly, Hetzner…) | **Alchemy v2** (Infrastructure as Effects) + distilled SDKs; `pocket deploy` target presets |
| **Edge runtime** (Workers + D1/R2/DO) | effect-cf, `@effect/sql-d1`, `@effect/sql-sqlite-do`; celld as a self-hosted DO runtime |
| **Durable workflows** | `unstable/workflow` (+ effect-temporal swap) |
| **AI features** (LLM tools over your data, MCP server) | `unstable/ai` (`LanguageModel`, `Tool`, `Toolkit`, `McpServer`) + `@effect/ai-*`; auto-generate MCP tools per collection |
| **Local-first / sync** | `unstable/eventlog` (+ encryption, SQL journal), foldkit-plus `Sync` |
| **Multi-tenancy** | `LayerMap.Service` (a Layer per tenant: DB, Storage) |
| **Distributed / scale-out** | `unstable/cluster` (entities, sharding, singletons) |
| **Type-safe RPC** alongside REST | `unstable/rpc` |
| **Testing** | `@effect/vitest`, `HttpApiTest` (in-memory client), `RpcTest`, test Layers for every service |
| **Caching** | `Cache`, `ScopedCache`, `PersistedCache`, UnJS ocache for HTTP |
| **Push notifications / chat widgets / slugs / QR / spreadsheets** | productdevbook family (nitroping, ahize, cizgile, etiket, hucre) and UnJS uqr |

**Takeaway:** Effect covers most of the **runtime** side (server, SQL, API, realtime primitives, jobs, cron, config, logging, rate limits, CLI, clients, testing). The **real product work** is in four places:
1. collection DSL and dynamic schemas
2. filter and rule language
3. admin UI
4. glue services wrapping non-Effect libraries (auth, mail, storage, thumbs, billing)

---

## 2. Guiding principles

1. **Contracts in Effect, implementations swappable.** Every capability is a `Context.Service` *interface* (a tag plus a Schema-typed contract), with no dependency on any vendor. Vendors live in separate adapter Layers. Swapping is a `Layer.provide` change and never touches app code. (Matches TWIE #135's direction: core `LanguageModel`/`Chat` became interface + Context.Service.)
2. **Schema is the single source of truth.** One collection definition derives the DB table, validation, API endpoints, OpenAPI, client types, admin forms (via annotations), MCP tools and test data (`Arbitrary`).
3. **Batteries included, not batteries welded.** `pocket` ships an opinionated **default preset** (`Pocket.layerDefault`: SQLite + better-auth + local fs + unemail SMTP + PersistedQueue). Every piece is replaceable, and the preset is just a composed Layer users can fork.
4. **Library first, binary second.** Users should be able to `import { Pocket } from "pocket"` inside their own Effect app (mount routes and services into an existing HttpRouter), *and* run `pocket serve` with zero code. PocketBase only offers the second; offering both is a key differentiator.
5. **Portable runtime.** Node/Bun first; Workers/D1 via Layers; avoid Node-only APIs in core (use `FileSystem`, `Path` services, never `node:fs`).
6. **Wrap Promise libraries at the edge only.** Non-Effect libs (better-auth, unemail, paykit, ipx, unstorage) get a thin adapter package each. Domain errors are Schema `TaggedError`s defined in the contract, not the vendor's error types.
7. **Pin to Effect v4 RC**, isolate `effect/unstable/*` usage behind our own services so upstream churn stays in adapters.

### What "not locked in" looks like concretely
```ts
// contract (packages/core) — no vendor imports
export class Mailer extends Context.Service<Mailer, {
  send(msg: MailMessage): Effect.Effect<MailReceipt, MailError>
}>()("pocket/Mailer") {}

// adapter (packages/mailer-unemail)
export const layerUnemail = (config: UnemailConfig) => Layer.effect(Mailer, Effect.gen(function*() {
  const client = createEmail(/* unemail driver */)
  return Mailer.of({
    send: Effect.fn("Mailer.send")(function*(msg) {
      return yield* Effect.tryPromise({ try: () => client.send(toUnemail(msg)), catch: (cause) => new MailError({ cause }) })
    })
  })
}))

// app — swap is one line
const App = Pocket.layer.pipe(Layer.provide(Mailer.layerUnemail(cfg)))   // or Mailer.layerMemory for tests
```
(Illustrative; verify unemail's exact API before implementing.)

---

## 3. Proposed architecture

### Layers of the system
```
┌───────────────────────────────────────────────────────────────┐
│ Apps: pocket CLI (serve/migrate/superuser/deploy) · Admin UI  │  Foldkit + foldkit-plus
├───────────────────────────────────────────────────────────────┤
│ Features: Records API · Realtime · Auth routes · Files · Batch│  HttpApi groups, generated per collection
│           Hooks bus · Jobs/Cron · Backups · Logs · MCP tools  │
├───────────────────────────────────────────────────────────────┤
│ Domain kernel: Collection DSL · Schema registry (dynamic) ·   │  pure Effect + Schema, no IO vendors
│   Filter/Rule language (parser → AST → SQL + evaluator) ·     │
│   Record service (CRUD + rules + hooks + events)              │
├───────────────────────────────────────────────────────────────┤
│ Service contracts (Context.Service interfaces):               │
│   Database · Auth · Storage · Mailer · Thumbnails · Queue ·   │
│   Realtime(PubSub) · Settings · Billing · Clock/Id · Logger   │
├───────────────────────────────────────────────────────────────┤
│ Adapters (Layers): sql-sqlite/pg/d1 (+Drizzle) · better-auth ·│
│   unstorage/fs/S3/R2 · unemail · ipx · PersistedQueue/effect-mq│
│   · paykit · redis/pg-listen pubsub · alchemy deploy targets  │
└───────────────────────────────────────────────────────────────┘
```

### Package layout (monorepo, pnpm/bun workspaces)
```
packages/
  core/              contracts, errors, Collection DSL, schema registry, filter/rule language, Record service
  server/            HttpApi definitions + handlers, realtime, batch, files, middleware (auth, rules, rate limit)
  client/            Effect SDK (HttpApiClient + realtime Stream) + Promise SDK facade
  cli/               effect/unstable/cli app: serve, migrate, superuser, typegen, deploy
  admin/             Foldkit admin SPA (served by server as static assets)
  preset-default/    Pocket.layerDefault composition
  adapter-sql-sqlite | adapter-sql-pg | adapter-sql-d1   (Drizzle effect-* based)
  adapter-auth-better-auth
  adapter-storage-fs | adapter-storage-unstorage
  adapter-mail-unemail
  adapter-thumbs-ipx
  adapter-queue-effect-mq
  adapter-billing-paykit
  deploy-alchemy     Alchemy stacks for CF (Workers+D1+R2) / AWS / VPS
  testing/           in-memory Layers for every contract + HttpApiTest helpers
examples/
docs/
```

### Two modes for collections
- **Code-first (typed):** `Collection.make("posts", { fields: { title: Field.text({ required: true }), author: Field.relation("users") }, rules: { list: "", create: "@request.auth.id != ''" } })`. Gives full TS types end to end.
- **Admin-first (dynamic, PocketBase-style):** definitions stored in DB as a `SchemaRepresentation` document, rebuilt at runtime with `fromRepresentation`, migrations generated from diffs. `pocket typegen` emits TS (`toCodeDocument`) so clients regain types.
Both compile to the **same internal `CollectionSpec`**. Build code-first first; dynamic mode reuses it.

### The hard, differentiating parts (do these deliberately)
1. **Filter/rule language.** Write the grammar (PB-compatible subset: `= != > >= < <= ~ !~ ?= ?~`, `&& ||`, parens, `@request.*`, `@collection.*`, datetime macros), parse to an AST (`Schema.TaggedUnion`), compile to Drizzle SQL, and add a pure evaluator for realtime filtering. Heavy property tests (`Arbitrary`) that the SQL result equals the in-memory evaluation.
2. **Record service pipeline:** decode → auth context → rule check → `before*` hooks → transaction (write + outbox event) → `after*` hooks → publish realtime. One `Effect.fn` pipeline; hooks and rules are services.
3. **Realtime correctness:** per-subscriber rule evaluation, backpressure (`Stream` buffering), reconnect/resume via event IDs.
4. **Admin UI:** generated forms from Schema annotations; foldkit-plus Surfaces to expose admin actions as MCP tools.

---

## 3a. Reference architecture: OpenCode v2 (Effect-based)
OpenCode (anomalyco/opencode, ~208k★) v2 is built on Effect, making it the largest real-world Effect app to study. Relevant patterns (from `packages/plugin/src/v2/effect/README.md` and the monorepo layout):
- **Monorepo split** we can mirror: `core`, `server`, `httpapi-codegen`, `protocol`, `schema`, `sdk-next`, `llm`, `session-ui`, plus private adapter packages `effect-drizzle-sqlite` (Drizzle + `@effect/sql-sqlite-bun`, with a migrator) and `effect-sqlite-node`. This is the same "Drizzle over @effect/sql" path recommended here.
- **Plugin model = scoped Effects.** `define({ id, effect: Effect.fn(function*(ctx) { ... }) })`. Plugins register hooks during setup; registrations are **owned by the plugin's Scope**, so closing the scope unregisters everything (hot reload/uninstall for free).
  - **Transform hooks** rewrite domain state (`ctx.agent/catalog/command/skill.transform`).
  - **Runtime hooks** intercept live operations (`ctx.aisdk.language(...)`).
  - `reload` re-runs transforms.
- **Dual plugin surface:** `/v2/effect` and `/v2/promise` entry points, so non-Effect authors can still write plugins. We should do the same (Effect-first API with a Promise facade).
- **Codegen from HttpApi** (`httpapi-codegen`) drives SDKs, the same plan as our `client` package.
- Note: its plugin package pins `effect@4.0.0-beta.83`; check how far its internals track RC before copying APIs.

**Adopt for pocket:** hooks/extensions = `Pocket.plugin({ id, effect })` with scope-owned registrations; transform hooks for collections/settings/routes; runtime hooks for record lifecycle, auth, mail and storage calls.

## 3b. API layer: one contract, pluggable transports
Define the API **once as Schema**, then expose it over any transport and serialization. Effect gives us this out of the box (names verified in rc.115 source).

**Two contract styles, both served:**
| | HttpApi | Rpc |
|---|---|---|
| Definition | `HttpApi` / `HttpApiGroup` / `HttpApiEndpoint` (paths, methods, status codes) | `Rpc.make` + `RpcGroup` (procedures, streaming, deferred responses) |
| Best for | Public REST, PocketBase-compatible routes, OpenAPI, non-TS clients, caching/CDN | First-party TS clients, admin UI, realtime streams, server↔worker, plugins, multi-node |
| Client | `HttpApiClient` (derived), OpenAPI → any language | `RpcClient` (derived), `AtomRpc` for UI |
| Middleware | `HttpApiMiddleware`, `HttpApiSecurity` | `RpcMiddleware` (+ ConnectionHooks) |
| Testing | `HttpApiTest` | `RpcTest` |

**Rpc transports are Layers:**
- **Server protocols** (`RpcServer`):
  - `layerProtocolHttp`
  - `layerProtocolWebsocket`
  - `layerProtocolSocketServer` (raw TCP/unix)
  - `layerProtocolStdio` (CLI/subprocess/MCP-style)
  - `layerProtocolWorkerRunner` (web/worker threads)
  - `toHttpEffect` / `toHttpEffectWebsocket`, to mount inside an existing HttpRouter
- **Client protocols** (`RpcClient`): `layerProtocolHttp`, `layerProtocolSocket` (WebSocket/TCP), `layerProtocolWorker`.
- **Serialization** (`RpcSerialization`): `layerJson`, `layerNdjson`, `layerJsonRpc`, `layerNdJsonRpc` (JSON-RPC 2.0 wire compat), `layerSchemaBinary` (compact binary).
- **Mounting:** `HttpRouter.toWebHandler` / `HttpApiBuilder` produce a web-standard `(Request) => Response`, so the same app runs on Node, Bun, Deno, Workers, or inside Nitro/h3/Hono/Elysia.

**Design for pocket:**
- `packages/protocol`: **all contracts** (`RecordsApi` HttpApi + `PocketRpc` RpcGroup) with shared Schema; no implementation.
- Handlers implemented once against the `Records` service; HttpApi and Rpc are thin adapters over it.
- A `Transport` choice is config: `pocket serve --rpc ws` / `http`; the admin UI uses Rpc over WebSocket (streams for realtime), public users get REST + SSE.
- Realtime: an Rpc streaming procedure (`subscribe(collection, filter)` → `Stream<RecordEvent>`) over WebSocket, **and** an SSE HttpApi endpoint for PocketBase-style clients. Both read the same `Realtime` service.
- Internal hops (server ↔ job workers, thumbnail worker, plugin sandboxes) use Rpc over Worker/Socket protocols. Scale-out uses `unstable/cluster` entities (built on Rpc).
- Third-party: expose a `JsonRpc` serialization for non-TS, and **MCP** via `unstable/ai` `McpServer` over stdio/HTTP reusing the same handlers.
- Custom transports: implement the `RpcClient`/`RpcServer` Protocol interface (e.g. over Durable Object WebSockets, MQTT, BroadcastChannel) as extra adapter packages.

## 3c. Deployment
Principle: **the app is a Layer; deployment is choosing a runtime Layer + infra.** Core never assumes a host.

| Target | How | Storage / DB | Realtime | Notes |
|---|---|---|---|---|
| **Single binary** (PocketBase-style) | `bun build --compile` with embedded admin assets; `NodeRuntime`/`BunRuntime.runMain` | SQLite file (`@effect/sql-sqlite-bun`), local fs | SSE/WS in-process PubSub | Primary target. Validate native SQLite + asset embedding in Phase 0 |
| **Docker / VPS** (Fly, Hetzner, Railway) | Node or Bun image; volume for data dir | SQLite or Postgres (`@effect/sql-pg` native client) | In-process; Postgres LISTEN/Redis PubSub for >1 instance | Alchemy v2 has Fly/Hetzner/Railway examples |
| **Cloudflare** | Workers via `HttpRouter.toWebHandler` + effect-cf; **Alchemy v2** provisions Worker, D1, R2, Queues, DO | D1 (`@effect/sql-d1`) or DO SQLite (`@effect/sql-sqlite-do`); R2 via unstorage/distilled | Durable Object per channel (WebSocket hibernation) | Requires realtime/storage Layers for CF; no local fs |
| **Self-hosted Workers** | Same Workers build on **celld** (Deno's distributed DOs from `wrangler.json`) | celld SQLite cells + your bucket | DO | Escape hatch from CF lock-in; unvalidated |
| **AWS** | Alchemy v2 (Lambda/ECS) + distilled AWS SDKs | RDS/Aurora Postgres, S3 | API Gateway WS or ECS long-lived | Later |
| **Embedded in a host app** | Import `Pocket.layer`, mount `toWebHandler` in Nitro/h3/Hono/Next/TanStack Start | host's choice | host's choice | Library mode |
| **Scale-out cluster** | `unstable/cluster` runners (HttpRunner/SocketRunner, K8sHttpClient) + `SqlMessageStorage` | Postgres | cluster-routed | Phase 3+ |

**Deployment package design:**
- `pocket deploy <target>` in the CLI delegates to `packages/deploy-alchemy` stacks (optional dependency). The stack is an Effect program, so it can read the same `pocket.config` and collection specs (e.g. provision R2 buckets for file fields).
- Runtime presets: `preset-node`, `preset-bun`, `preset-cloudflare` = Layers choosing DB/Storage/PubSub/Realtime adapters for that host.
- Ops built in: `/api/health`, OTLP export (`unstable/observability`), graceful shutdown via Scope finalizers, backups to Storage, migrations run on boot (flag).
- CI: build the binary + Docker image per release; Alchemy `plan`/`deploy` in CI for CF examples.

## 3c-bis. Email: templates + delivery
Split into two contracts so rendering and sending swap independently:
- **`Mailer`** (delivery): unemail Layer (SMTP, Resend, SES, Postmark…), memory/preview Layer for dev/tests.
- **`EmailTemplates`** (rendering): `render(template, props) → Effect<{ html, text, subject }, TemplateError>`, with props typed by Schema.
  - **Default renderer:** **jsx-email** (`jsx-email@3.2.1`, shellscape/jsx-email, 1.3k★; React 19 peer; plugins for inline CSS, minify, pretty; `canispam` checks). Alternative Layers: React Email (`@react-email/render@2.1.0`), MJML.
  - **Component library:** **emailcn** (shadcn-labs/emailcn, 286★, MIT, emailcn.run): shadcn-registry email blocks (marketing bento grids, stats, receipts, auth emails) for React Email, MJML React and JSX Email. Installed via `npx shadcn add <registry-url>`, so users **own the template source** (Rails-generator feel).
- **pocket integration:**
  - `pocket g email <name>` scaffolds a jsx-email template with a Schema props contract.
  - Built-in templates: verification, password reset, OTP/magic link, invite (better-auth), receipts/invoices/dunning (PayKit), usage/stats digests (bento stats grid) sent by cron or workflows.
  - Admin: template preview with sample props (generated via `Arbitrary`), test-send, delivery log (traced).
  - Sending is a job by default (`Jobs.enqueue(SendEmail)`) with retries; workflows can `yield*` send steps durably.
- **Note:** React is only a server-side render dependency for emails (HTML string out), independent of the Foldkit admin UI. Keep it in the `email-jsx` adapter package so apps without email don't pull React.

## 3d. Observability as a built-in feature
Effect traces, logs and metrics natively, so pocket gets deep observability almost for free and can sell it.
- **Automatic spans:** every `Effect.fn("Records.create")`, HttpApi/Rpc handler, SQL statement (`@effect/sql`), HttpClient call, job, workflow activity and cluster message produces spans with parent/child context. Trace context propagates across HTTP (`HttpTraceContext`), Rpc and cluster hops.
- **Structured logs** (`Logger`, `Effect.annotateLogs`) are correlated to spans; **metrics** via `Metric` (counters, histograms, gauges).
- **Export:** `effect/unstable/observability` (`Otlp`, `OtlpTracer`, `OtlpLogger`, `OtlpMetrics`, `PrometheusMetrics`) is lightweight with no OTel SDK dependency; env-var configured Layer. Or `@effect/opentelemetry` to join an existing OTel setup. Works with Grafana/Tempo, Honeycomb, Datadog, Axiom, SigNoz, Jaeger.
- **Built into pocket:**
  - Default spans and attributes for collection, record id, rule result, user, tenant, plan, job and workflow id.
  - **Admin "Observability" screen:** a built-in lightweight trace/log store (SQLite table fed by a local exporter Layer) so the single binary has a trace viewer with no external stack, PocketBase-logs++.
  - Dev mode: `pocket dev` can pipe to **motel** (local OTel TUI) or Effect DevTools (VS Code extension).
  - `/metrics` Prometheus endpoint; health/readiness endpoints.
  - Redaction by default (`Redacted`, Schema `Redacted` fields never logged).
- **Swappable:** exporter choice is a Layer (`Observability.layerLocal`, `layerOtlp`, `layerPrometheus`, `layerNone`).

## 3e. Runtime independence (Node, Bun, Deno, Workers, browser)
Effect v4 abstracts the platform behind services, so pocket's core should import **no runtime-specific modules**.
- **Abstract services in core `effect`:** `FileSystem`, `Path`, `Terminal`, `Stdio`, `Crypto`; `unstable/http` `HttpServer`/`HttpClient`; `unstable/socket` `Socket`/`SocketServer`; `unstable/process` `ChildProcessSpawner`; `unstable/workers` `Worker`; `KeyValueStore`; `Clock`/`Random`.
- **Platform Layers provide them** (all at 4.0.0-rc.115): `@effect/platform-node`, `@effect/platform-bun`, `@effect/platform-deno`, `@effect/platform-browser`; Workers via `HttpRouter.toWebHandler` + effect-cf.
- **Web-standard handler:** the whole app compiles to `(Request) => Response`, so it also runs inside Nitro/h3/srvx, Hono, Elysia and Next.
- **Runtime-specific pieces stay in presets:**

| Concern | Node | Bun | Deno | Workers |
|---|---|---|---|---|
| Runtime entry | `NodeRuntime.runMain` | `BunRuntime.runMain` | platform-deno | `toWebHandler` export |
| SQLite | `@effect/sql-sqlite-node` (`node:sqlite`) | `@effect/sql-sqlite-bun` | libsql / wasm | D1 / DO SQLite |
| Postgres | `@effect/sql-pg` (native protocol, no `pg` dep) | same | same | via Hyperdrive/socket (verify) |
| Files | `FileSystem` (node) | `FileSystem` (bun) | `FileSystem` (deno) | R2 Storage Layer |
| Realtime | in-process PubSub | same | same | Durable Objects |
| Single binary | Node SEA (verify) | `bun build --compile` | `deno compile` | n/a |

- **Rules for contributors:** no `node:*`/`Bun.*`/`Deno.*` imports outside `preset-*` and adapter packages (enforce with lint and `impound`); CI matrix runs the test suite on Node, Bun and Deno, plus a Workers smoke test (miniflare/wrangler).
- **Pitch:** "write once, run on Node, Bun, Deno or the edge; choose at deploy time."

---

## 4. Key decisions (recommendations; open to challenge)

| Decision | Recommendation | Why / alternative |
|---|---|---|
| Query layer | **Drizzle `effect-*`** on top of `@effect/sql-*` clients | Typed query builder, multi-dialect, effect-schema derivation, already v4. Alternative: raw `SqlClient` + `SqlSchema` (fewer deps, more hand-written SQL). Dynamic collections may need raw SQL anyway: keep both. |
| API transport | **Both, from one `protocol` package:** HttpApi (REST + SSE, OpenAPI) for public and PocketBase-compatible use; Rpc (WebSocket/HTTP, NDJSON or binary) for admin, SDK streams and internal hops | Transports and serialization are Layers (§3b), so this costs little and stays swappable. |
| Plugin model | OpenCode-v2-style `define({ id, effect })` with scope-owned hook registrations + Promise facade | Proven at scale in an Effect app; clean unload/reload. |
| Deployment | Single binary first; runtime presets as Layers; Alchemy v2 optional package for CF/AWS/VPS | See §3c. |
| Auth | **better-auth behind `Auth` contract** | Most complete feature set; Effect community adapters are v3 so write our own thin adapter. Keep sessions and tokens abstract so a native Effect auth can replace it later. |
| Realtime transport | **SSE first** (PB-compatible), WebSocket (crossws / `SocketServer`) later | SSE works through proxies and matches PB clients. |
| Default DB | **SQLite** (`@effect/sql-sqlite-node` uses `node:sqlite`; bun variant) | PocketBase parity: zero-ops single file. Postgres for scale. |
| Frontend (admin) | **Foldkit + foldkit-plus** | Same Schema/Effect mental model; Surfaces and Agent give MCP admin. Risk: young ecosystem (pinned exact to rc.115). Alternative: Solid + `@effect/atom-solid`. |
| Client state for users' apps | Ship `@effect/atom-*` helpers (`AtomHttpApi`) and a Foldkit integration | Don't force Foldkit on end users. |
| Jobs | `PersistedQueue` default, effect-mq adapter | Official and zero-dep default; effect-mq when Postgres/Redis present. |
| Files | Own `Storage` contract; unstorage adapter covers fs/S3/R2 | unstorage is Promise-based but broad; distilled for native R2/S3 later. |
| Deploy | Alchemy v2 stacks as an optional package | Keep core deploy-agnostic (Docker/binary works without Alchemy). |
| Runtime | Bun + Node supported; Workers as a later target | Workers constrains FS/SQLite/long-lived SSE (needs DO). |
| Versioning | Pin `effect@4.0.0-rc.115` exactly across workspace; one bump PR at a time | Foldkit and effect-temporal pin exact RCs; mismatches break types. |

---

## 4a. Product direction: all-out (decided 2026-09-16)
Scope is **full-stack "Rails power + Effect flexibility"**. Billing (PayKit), deploy (Alchemy), jobs (effect-mq / PersistedQueue), durable workflows, cluster, AI/MCP and realtime are **core selling points**, not add-ons. How to deliver that scope without it collapsing:

1. **Rails-style conventions on top of Effect flexibility.** Every pillar ships a *blessed default* and a *generator*. `pocket new`, `pocket g collection|job|workflow|plan|entity|rpc|page` produce idiomatic code. Swapping adapters is possible but never required to get started.
2. **One app definition drives every pillar.** `pocket.config.ts` + collection specs feed DB, API, admin, billing entitlements (plans gate rules: `@request.auth.plan.feature("export")`), jobs (typed payloads), workflows, cluster entities and Alchemy infra (buckets, queues, DBs derived from what the app uses).
3. **Integrated, not bundled.** Pillars must *know about each other*:
   - **Rules** can reference auth, billing entitlements and tenancy.
   - **Record hooks** can enqueue jobs or start workflows.
   - **Workflows** can send mail, charge, and wait on realtime or durable events.
   - **The admin UI** shows jobs, workflow runs, cluster shards, subscriptions and deploy state.
   - **MCP** exposes all of it.

   That cross-wiring is the Rails magic, and it is our product.
4. **Stability tiers, published.** `stable` (core, collections, rules, auth, API, realtime), `preview` (billing, workflows, deploy), `experimental` (cluster, sync). Every pillar sits behind our contracts so upstream RC churn stays in adapters.
5. **Vertical-slice milestones, each pillar end to end.** Each slice ships: contract, default adapter, test Layer, generator, admin screen, docs page and example app. No pillar is "done" without all seven.
6. **Flagship example apps as the integration test suite:** a SaaS starter (auth + orgs + billing + jobs + deploy to CF), a realtime collab app (sync + realtime + cluster), an AI agent app (workflows + ai + MCP). If the examples are easy to write, the framework works.
7. **Docs are a first-class package** (guides per pillar, "swap this adapter" recipes, v4-accurate); ship agent skills (`pocket` skill for Effect-TS/skills-style agents) since agents are a primary user.
8. **Two audiences, one core:** the zero-code path (`pocket serve`, admin, REST, Promise SDK) for adoption, and the Effect power path for building real apps. Both hit the same services.

## 5. Phased plan
> Superseded in emphasis by §4a: phases below remain the build order, but every pillar in Phase 3 is core product, scheduled as vertical slices rather than optional extras.

**Phase 0 — Spike (1–2 weeks)**
- Monorepo scaffold, pinned deps, `@effect/vitest`, `@effect/tsgo` LSP, Effect-TS/skills installed for agents.
- Prove the risky integrations end to end in one file: Drizzle `effect-sqlite` + HttpApi + better-auth handler mounted + SSE stream + Foldkit page calling `HttpApiClient`.

**Phase 1 — Kernel (MVP "PocketBase-lite")**
- Contracts + in-memory test Layers.
- Code-first Collection DSL → Drizzle table + Schema + CRUD HttpApi group.
- Filter/sort/pagination/expand; rule language v1 (list/view/create/update/delete).
- better-auth adapter (email/pw + OAuth), superuser bootstrap.
- Realtime SSE with rule filtering.
- Files (fs) + ipx thumbs; unemail mailer.
- CLI: `serve`, `migrate`, `superuser`, `typegen`.

**Phase 2 — Admin & dynamic collections**
- Foldkit admin: collections editor, records browser, settings, logs.
- `SchemaRepresentation`-backed dynamic collections + migration diffing.
- Hooks bus; custom routes; cron + PersistedQueue; backups; rate limiting; request logs.
- Promise SDK + Effect SDK published.

**Phase 3 — Beyond PocketBase**
- Postgres/D1 adapters, S3/R2 storage, effect-mq, PayKit billing, Alchemy deploy targets (CF Workers + D1 + R2 + DO for realtime).
- MCP server per collection (`unstable/ai` McpServer + foldkit-plus Agent).
- Multi-tenancy via `LayerMap`; eventlog/local-first sync; cluster scale-out.

---

## 6. Risks & open questions
- **Effect v4 is RC; `unstable/*` can break in minors.** Mitigation: contracts wrap unstable APIs; pin exact versions.
- **Drizzle Effect is also RC** (1.0.0-rc.x), and dynamic collections fit a static query builder poorly. Decide the raw-SQL boundary early.
- **better-auth's own schema and migrations** vs our migration system: who owns the users table? (Recommend: better-auth tables managed by its CLI, and our "auth collections" extend the user via a 1:1 profile table.)
- **Foldkit maturity** (0.160, exact rc pin) and bundle/SSR needs for the admin.
- **PB API compatibility**: aim for wire-compat with the PocketBase JS SDK? It's a huge adoption lever but constrains design. Decide in Phase 0.
- **Single binary**: `bun build --compile` + embedded admin assets + SQLite native module. Validate early.
- **Workers target** constrains long-lived realtime (needs Durable Objects) and file system use.
- Licensing and naming.

## 7. Immediate next steps
1. Decide: PocketBase API wire-compatibility yes/no; code-first vs dynamic priority; Bun vs Node default.
2. Run the Phase 0 spike (§5).
3. Write the contract signatures for `Database`, `Auth`, `Storage`, `Mailer`, `Realtime`, `Hooks`, `Records` as the first real code, before any adapter.
4. Write the filter/rule grammar spec (PB-compatible subset) as a doc and test corpus.
