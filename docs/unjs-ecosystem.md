# UnJS Ecosystem (researched 2026-09-16)

UnJS (https://unjs.io, led by Pooya Parsa / pi0) makes small, runtime-agnostic JS/TS libraries that run on Node, Deno, Bun, browsers and edge workers. They're the base layer under Nuxt and Nitro. Sources: GitHub API for the `unjs`, `h3js` and `nitrojs` orgs and for `productdevbook`, plus GitHub search. Star counts and "last push" dates are as of 2026-09-16. Every official repo listed was pushed within the last month unless a date is given.

Note: H3 and Nitro moved into their own orgs (`h3js`, `nitrojs`). Retired projects sit in `unjs-archive`: lmify was replaced by nypm, and unkit was the standard library.

## Server & HTTP
| Package | ★ | What |
|---|---|---|
| nitro (nitrojs/nitro) | 11.2k | Server toolkit: build once and deploy to any provider through presets (Node, CF, Vercel, Deno, Bun…) |
| h3 (h3js/h3) | 5.4k | Minimal, portable HTTP framework built on web standards (v2) |
| srvx (h3js) | 865 | Universal `serve()` on web-standard Request/Response for Node, Deno and Bun |
| rou3 (h3js) | 745 | Lightweight, fast router (was radix3) |
| crossws (h3js) | 731 | Cross-runtime WebSocket servers (Node/Deno/Bun/CF). pi0/y-crossws adds a Yjs server |
| rendu (h3js) | 178 | "JavaScript Hypertext Preprocessor", PHP-style templates |
| listhen | 589 | Polished HTTP listener for dev (HTTPS, tunnel, QR code) |
| httpxy | 345 | Full-featured HTTP and WebSocket proxy |
| get-port-please | 299 | Find an available port |
| untun | 1.4k | Expose localhost through Cloudflare Quick Tunnels |
| serve-placeholder | 170 | Placeholder responses for missing assets |
| cookie-es | 255 | Cookie / Set-Cookie parse and serialize |
| node-mock-http | 17 | Mock Node HTTP req/res |
| nitro-cloudflare-dev (nitrojs) | 150 | CF bindings in the Nitro/Nuxt dev server |
| ocache | 127 | Composable caching: TTL, SWR, HTTP response caching on standard Request/Response |

## Fetch & APIs
| Package | ★ | What |
|---|---|---|
| ofetch | 5.4k | Improved fetch: auto JSON, retries, interceptors, works everywhere |
| node-fetch-native | 189 | Native-first fetch polyfill |
| ufo | 1.3k | URL utilities |
| fetchdts | 126 | Type utilities for strongly typed fetch APIs |
| ungh | 687 | Unlimited-rate GitHub API proxy (ungh.cc) |
| openapi-renderer | 49 | Renders an OpenAPI spec to HTML |

## Storage, data, serialization
| Package | ★ | What |
|---|---|---|
| unstorage | 2.6k | Async key-value API with dozens of drivers (fs, redis, CF KV, S3…), mounting and watching |
| db0 | 357 | Lightweight SQL connector (sqlite, pg, mysql, D1, libsql, bun…) with Drizzle integration |
| destr | 1.4k | Safe, fast alternative to JSON.parse |
| ohash | 732 | Object hashing, serialization, diffing |
| confbox | 301 | YAML/TOML/JSONC/JSON5/INI parse and stringify |
| undio | 243 | Convert between JS data types (streams, buffers, blobs…) |
| capnp-es | 177 | Cap'n Proto serialization in TypeScript |
| nanotar | 199 | Tiny tar utilities for any runtime |
| uncrypto | 259 | One Web Crypto API across runtimes |
| defu | 1.4k | Recursive default assignment |

## Config, env, runtime
| Package | ★ | What |
|---|---|---|
| c12 | 897 | Smart config loader (rc, env-specific config, extends, remote layers) |
| rc9 | 313 | Read and write rc files |
| std-env | 641 | Detects runtime, provider, CI and environment |
| unenv | 769 | Node.js compatibility layer for non-Node runtimes (used by CF Workers/Wrangler) |
| runtime-compat | 275 | Table of API compatibility across runtimes |
| env-runner | 48 | Generic runner for JS runtime environments |
| compatx | 63 | Compatibility-date toolkit |
| unctx | 603 | Composables / async context in plain JS |
| hookable | 960 | Awaitable hook system |
| perfect-debounce | 342 | Debounce for async functions |

## Build & modules
| Package | ★ | What |
|---|---|---|
| unplugin | 3.6k | One plugin API for Vite, Rollup, webpack, esbuild, Rolldown, Rspack |
| jiti | 3.0k | Runtime TS/ESM loader for Node |
| unbuild | 2.7k | Unified build system (rollup and mkdist) |
| obuild | 432 | Zero-config ESM/TS package builder (rolldown); successor direction to unbuild |
| mkdist | 437 | File-to-file transpiler |
| unimport | 682 | Auto-import engine (powers Nuxt auto-imports) |
| magicast | 2.5k | Programmatically modify JS/TS source (config codemods) |
| knitwork | 319 | Generate safe JS code strings |
| mlly | 529 | ESM module utilities |
| exsolve | 84 | Module resolution based on Node's implementation |
| pkg-types | 300 | package.json and tsconfig types and utilities |
| unwasm | 305 | WebAssembly import tooling |
| impound | 88 | Restrict import patterns in parts of a codebase |
| nf3 | 72 | Trace and copy only the node_modules needed at runtime |
| untyped | 527 | Generate types and markdown from a config schema |
| unrouting | 195 | Universal filesystem routing |
| webpackbar | 2.1k | Webpack progress bar and profiler |

## CLI, DX, project tooling
| Package | ★ | What |
|---|---|---|
| consola | 7.3k | Polished console logger and prompts |
| citty | 1.3k | CLI builder |
| nypm | 708 | Unified package manager API (npm/pnpm/yarn/bun/deno) |
| jup (+ setup-jup) | 66 | Pin and run the right package manager or runtime per project |
| giget | 773 | Download templates and git repos |
| changelogen | 1.3k | Changelogs from conventional commits |
| automd | 336 | Keeps markdown (README) sections updated automatically |
| codeup | 58 | Automated codebase updater (POC) |
| undocs | 364 | Docs theme and CLI used across UnJS |
| template | 176 | UnJS project starter |
| scule | 515 | String case utilities |
| magic-regexp | 4.3k | Type-safe, readable RegExp that compiles away |
| pathe | 589 | Normalized drop-in for `path` |
| untracing | 27 | Naming registry for tracing channels (WIP, last push 2026-03) |

## Content, media, frontend
| Package | ★ | What |
|---|---|---|
| unhead | 1.3k | Full-stack `<head>` manager for any framework (v3) |
| ipx | 2.5k | Image optimizer (powers Nuxt Image) |
| image-meta | 136 | Detect image type and size |
| unpdf | 1.2k | PDF text extraction and rendering in any runtime (last push 2026-08) |
| md4x | 412 | Fast, small markdown parser and renderer |
| mdbox | 126 | Simple markdown utilities |
| uqr | 758 | QR codes to ANSI, Unicode or SVG (last push 2026-04) |
| fontaine | 2.0k | Font fallback metrics to reduce CLS |
| unifont | 278 | Access font CDNs and providers |
| theme-colors | 269 | Generate color shades |

## Community "un-style" libraries (universal, zero-dependency)

**productdevbook** (Mehmet, maker of unemail) has a large zero-dependency, runtime-agnostic family:
| Package | ★ | What |
|---|---|---|
| [unemail](https://github.com/productdevbook/unemail) | 291 | One email API across 18 providers (SMTP, Resend, SES, Postmark, SendGrid, Mailgun…), RFC 8058 and DKIM, edge-first |
| [unadapter](https://github.com/productdevbook/unadapter) | 59 | Type-safe database adapter layer over Drizzle, Prisma, Kysely, Knex, Mongo, Sumak, memory |
| [sumak](https://github.com/productdevbook/sumak) | 127 | AST-first, hookable, type-safe SQL query builder |
| [misina](https://github.com/productdevbook/misina) | 77 | Driver-based fetch client: retry, RFC 9111 cache, cookies, circuit breaker, SSE/NDJSON, OpenAPI types (last push 2026-05) |
| [silgi](https://github.com/productdevbook/silgi) | 35 | End-to-end type-safe RPC with compiled pipelines (last push 2026-04) |
| [hucre](https://github.com/productdevbook/hucre) | 2.2k | Spreadsheet engine: XLSX/CSV/ODS read and write |
| [etiket](https://github.com/productdevbook/etiket) | 458 | Barcode and QR generator, 40+ formats |
| [cizgile](https://github.com/productdevbook/cizgile) | 99 | URL slug engine (RFC 3986/3987, transliteration) |
| [ahize](https://github.com/productdevbook/ahize) | 39 | One API for 18 live-chat widgets (last push 2026-04) |
| [portakal](https://github.com/productdevbook/portakal) | 63 | Printer-language SDK (ZPL, ESC/POS, TSC…) (last push 2026-04) |
| [pencere](https://github.com/productdevbook/pencere) | 46 | Lightbox with View Transitions (last push 2026-06) |
| [seslen](https://github.com/productdevbook/seslen) | 27 | Web Audio UI-sound library (last push 2026-04) |
| [dokuma](https://github.com/productdevbook/dokuma) | 7 | Headless UI primitives (last push 2026-04) |
| nitroping (+ nitroping-push, nitroping-sdk) | 7 / 338 / 24 | Feedback, push and audit-log platform on Cloudflare, Alchemy **and Effect**. nitroping-push last push 2026-04 |
| [nitro-graphql](https://github.com/productdevbook/nitro-graphql) | 127 | GraphQL module for Nitro with type generation (last push 2026-03) |
| [port-killer](https://github.com/productdevbook/port-killer) | 5.1k | Desktop port-management tool (last push 2026-07) |

**Other community projects around UnJS:**
- johannschopplich/apiful (102★): composable fetch client typed from OpenAPI with no codegen. It succeeds unrested.
- harlan-zw: unhead maintainer, unlighthouse.
- Intevel/h3-valibot (78★): Valibot validation for h3 (last push 2025-11).
- pi0/h3-on-edge (65★): streaming edge workers on h3 (last push 2026-04).
- profilecity/unstorage-s3-driver: S3 driver for unstorage (last push 2024-08).
- barelyhuman/nitro-preact-islands: Preact islands for Nitro (last push 2026-03).
- una-ui/una-ui (677★): Nuxt UI framework on UnoCSS. It shares the "un" name but isn't UnJS.
- Barbapapazes/unjs-relations: graph of dependencies between UnJS packages (last push 2024-03).

## Overlap with Effect (see effect-ecosystem.md)
- UnJS gives you lightweight Promise-based building blocks. Effect has its own versions: ofetch ↔ `HttpClient`, unstorage ↔ `KeyValueStore`, db0 ↔ `@effect/sql-*`, citty ↔ `unstable/cli`, consola ↔ `Logger`, hookable ↔ `PubSub`.
- The two combine well at the edges. Nitro or h3 can host Effect handlers through `ManagedRuntime`, and unstorage or unemail can be wrapped as Effect services.
- productdevbook's nitroping is an example of an app that mixes UnJS-style tooling with Effect and Alchemy.
