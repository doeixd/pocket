# Shadcn Labs Ecosystem (researched 2026-09-16)

https://www.shadcn-labs.com: "Pushing the limits of the shadcn/ui ecosystem." Run by **Aniket Pawar** (sponsorship $499/mo, claims 50K+ monthly visitors). Every project is a **shadcn registry**: install with `npx shadcn@latest add <registry-url>/<item>` and the source is copied into your repo (you own it). MIT, "100% free, zero config". Stars/dates from GitHub API 2026-09-16.

## Projects
| Project | ★ | Site | Built on | What |
|---|---|---|---|---|
| **pdfcn** | 1.8k | pdfcn.dev | Takumi, Forme | PDF components for React (invoices, reports, documents) |
| **termcn** | 1.1k | termcn.dev | Ink, OpenTUI | Terminal UI components for React |
| **agentcn** | 469 | agentcn.run | Eve, Flue | "shadcn/ui for building agents": production-ready AI agent recipes |
| **editorcn** | 303 | editorcn.vercel.app | Tiptap | Rich text editor components |
| **emailcn** | 286 | emailcn.run | React Email, MJML React, JSX Email | Email components/blocks (auth, receipts, marketing bento grids…) |
| **ogimagecn** | 227 | ogimagecn.com | Satori | Open Graph image components |
| **framecn** | 139 | framecn.dev | Editframe | Video components |
| **shadcn-cssinjs** | 118 | shadcn-cssinjs.com | StyleX | CSS-in-JS port of shadcn/ui |
| **shadercn** | 63 | shadercn.run | vgpu, TypeGPU | Shader (WebGPU) components |
| **mcpcn** | 44 | mcpcn.dev | Base UI | UI components for ChatGPT/Claude/MCP apps |
| **startercn** | 36 | startercn.vercel.app | shadcn registry | Registry template: landing page, docs, agent support, haptics/audio/animations |
| **skills** | 22 | skills.sh/shadcn-labs/skills | — | Agent skills for shadcn registries |
| slidecn | 5 | slidecn.vercel.app | reveal.js | Presentation components |
| shadcn-lynx | 3 | shadcn-lynx.com | Lynx | shadcn/ui for Lynx (ByteDance cross-platform) |
| outbid-template | 24 | — | — | Site template |
| awesome-{flue,eve,langgraph,mastra}-agents | 10/8/2/1 | — | — | Curated agent lists |
| shadcnweekly.com | 0 | shadcnweekly.com | — | Weekly shadcn newsletter |

All are React-based (except shadcn-lynx). **None use Effect.**

## Relevance to pocket (app-in-a-box)
The registry model matches pocket's **Rails-generator philosophy**: generate owned source, not opaque deps. Fits by pillar:

| pocket feature | Shadcn Labs project | How |
|---|---|---|
| Transactional + marketing email | **emailcn** (+ jsx-email) | Default template blocks behind `EmailTemplates` contract (see plan §3c-bis) |
| Invoices, receipts, reports, exports | **pdfcn** | `Documents`/`PdfRenderer` contract; PayKit invoices → PDF; `pocket g pdf <name>`; render in a job |
| Social/OG images for records & pages | **ogimagecn** (Satori) | `OgImages` service: `/api/og/:collection/:id` rendered + cached in Storage |
| Rich text fields | **editorcn** (Tiptap) | Rich-text field type editor for user apps (admin is Foldkit/non-React; would need a web-component wrapper or Tiptap directly) |
| CLI / TUI (`pocket` dev console, logs, jobs dashboard) | **termcn** (Ink/OpenTUI) | `pocket dev` TUI: jobs, workflow runs, traces; alternative: effect-boxes / motel style |
| AI agents over app data | **agentcn** (Eve/Flue), **mcpcn** | Agent recipes, MCP app UI; pocket's core stays `unstable/ai` + McpServer, agentcn as example recipes |
| Starter/docs site for pocket itself or user apps | **startercn** | Template for a pocket components registry and docs |
| Agent skills | **skills** | Model for shipping a `pocket` skills pack |
| Video / shaders / slides | framecn, shadercn, slidecn | Out of scope; possible marketing/templates |

### Idea: a pocket registry
Publish **pocket's own shadcn-compatible registry** (via startercn) so generators and UI blocks are installable the same way. Examples: `npx shadcn add pocket/auth-forms`, `pocket/billing-pricing-table` (PayKit), `pocket/realtime-list` (Atom + Rpc), `pocket/email-verify` (emailcn-based). This gives Rails-style scaffolding for **user frontends** in React while pocket's admin stays Foldkit.

### Caveats
- Single maintainer across ~20 projects; star counts modest outside pdfcn/termcn. Treat as **copy-in templates**, not runtime dependencies (the registry model already implies this, so risk is low).
- React-only: renderers (email/PDF/OG) run server-side and should live in adapter packages so core stays React-free.
- Underlying engines matter more than the blocks: jsx-email/React Email/MJML, Takumi/Forme, Satori, Tiptap, Ink/OpenTUI. Check each engine's runtime support (Workers/Bun) before making it a default.
