# Unpic Ecosystem & Integration with pocket / Foldkit / foldkit-plus

Researched 2026-09-16. Sources: https://unpic.pics, GitHub (ascorbic/*), npm packages inspected via `npm pack` (type declarations and READMEs). By **Matt Kane** (ascorbic), MIT. Related: [pocketbase-map-and-architecture.md](pocketbase-map-and-architecture.md) (Files & Thumbnails), [shadcn-labs-integration.md](shadcn-labs-integration.md), [foldkit-bundle-implementation-plan.md](foldkit-bundle-implementation-plan.md), [unjs-ecosystem.md](unjs-ecosystem.md) (ipx).

## 1. What Unpic is
"The best images for every framework." Use **your existing image CDN's URL API** to deliver responsive, modern-format images, instead of build-time or server-side resizing. Just an `<img>` with a correct `srcset`/`sizes`, lazy loading, async decoding, layout-shift-free sizing and optional placeholders. No wrappers, no runtime JS.

## 2. Packages (verified)
| Package | Version | Deps | Role |
|---|---|---|---|
| **`unpic`** (lib; JSR `@unpic/lib`) | 4.2.2 | none | **Universal image CDN URL translator.** Detect the CDN from a URL, parse existing transforms, generate new transform URLs |
| **`@unpic/core`** | 1.0.3 | `unpic` | Framework-agnostic image component logic: `transformBaseImageProps`, `transformSharedProps`, `transformBaseSourceProps` (for `<picture>`), `getSrcSet`, `getSrcSetEntries`, `getBreakpoints`, `getSizes`, `getStyle`, `normalizeImageType`, `DEFAULT_RESOLUTIONS`; entry points `.` and `./base` |
| `@unpic/react`, `vue`, `solid`, `svelte`, `astro`, `preact`, `qwik`, `angular`, `lit`, `webc` | 1.0.x | `@unpic/core` | Thin framework components (ascorbic/unpic-img, **2.1k★**, pushed 2026-09-16) |
| **`@unpic/placeholder`** | 0.1.2 | `blurhash` | LQIP: `getDominantColor`, `getPalette`, `kMeansClusters`, `blurhashToCssGradients`/`…String`, `blurhashToDataUri`, `blurhashToImageCssObject`/`…String`, `pixelsToCssGradients`, `rgbaPixelsToBmp`, `imageDataToDataURI`, `rgbColorToCssString`. Works on Node, Deno and edge runtimes (ascorbic/unpic-placeholder, 188★) |
| **`@unpic/pixels`** | 1.3.0 | `pngjs`, `jpeg-js` | Decode PNG/JPEG to raw pixels + dimensions: `getPixels(url \| buffer)`, `getFormat`, `decodeImageData`, `getDataFromUrl` (Node) |

**unpic lib API [V]**
- **Transform:** `transformUrl({ url, provider?/cdn?, fallback?, width, height, format, quality }, providerOperations?, providerOptions?) → string | undefined`; `getTransformerForCdn`.
- **Detect:** `getProviderForUrl(url) → ImageCdn | false`, plus `…ByDomain` / `…ByPath`.
- **Extract:** `parseUrl(url, cdn?, options?)`, `getExtractorForUrl`, `getExtractorForProvider`, `parsers`.
- **Async lazy provider loading (`unpic/async`):** `getModuleForProvider`, `getGeneratorForProvider`, `getTransformerForProvider`, async `transformUrl`.
- **Per-provider modules** export `generate`, `extract`, `transform`.

**Supported providers (33):** appwrite, astro, builder.io, bunny, **cloudflare**, **cloudflare_images**, cloudimage, cloudinary, contentful, contentstack, directus, hygraph, imageengine, imagekit, imgix, **ipx**, keycdn, kontent.ai, **netlify**, nextjs, scene7, shopify, storyblok, **supabase**, uploadcare, **vercel**, wordpress, wsrv.

## 3. Why it fits pocket
pocket's plan has a **Files/Storage** service (local fs, S3/R2), **`Thumbnails` via UnJS ipx**, OG images, emails and a Foldkit admin. Unpic unifies the *delivery* side:

| Concern | Without unpic | With unpic |
|---|---|---|
| Thumbnails for file fields | pocket-specific `?thumb=100x100` endpoint (PocketBase style) | Same ipx endpoint, but **URL generation is standard**: `transformUrl({ url, cdn: "ipx", width, height, format })` |
| Deploy on Cloudflare | Custom R2 + resize code | Switch the transform provider to `cloudflare` (Image Resizing) or `cloudflare_images`; **app code unchanged** |
| User-supplied or CMS image URLs (Shopify, Cloudinary, Contentful…) | Download and re-process, or serve as-is | Detect the CDN and use *its* transform URL API |
| Responsive markup | Hand-written srcset/sizes per component | `@unpic/core` computes srcset/sizes/style for fixed/constrained/fullWidth layouts |
| Placeholders | none | Compute BlurHash / dominant color **on upload** (server), store in record metadata, render CSS gradients instantly |

## 4. Integration design

### 4.1 Server (Effect services)
```ts
// Contract: pocket/Images
class Images extends Context.Service<Images, {
  /** Delivery URL for a stored file or any external URL, via the configured provider (or detected CDN). */
  url(source: ImageSource, ops: ImageOps): Effect.Effect<string, ImageUrlError>
  /** Full responsive attribute set (src, srcset, sizes, style, width, height, loading, decoding, fetchpriority). */
  attributes(source: ImageSource, props: ResponsiveImageProps): Effect.Effect<ImageAttributes, ImageUrlError>
}>()("pocket/Images") {}

// Contract: pocket/ImageMetadata (computed at upload time)
class ImageMetadata extends Context.Service<ImageMetadata, {
  analyze(file: StoredFile): Effect.Effect<{ width; height; format; dominantColor; blurhash?; palette? }, ImageAnalysisError>
}>()("pocket/ImageMetadata") {}
```
- **Adapters (Layers)**
  - `Images.layerUnpic({ provider: "ipx" | "cloudflare" | "vercel" | "netlify" | "supabase" | …, baseUrl, providerOptions })`: wraps `transformUrl` (sync, pure) and `@unpic/core` `transformBaseImageProps`. Provider choice follows the deploy preset (plan §3c): single binary/Docker → `ipx` (pocket serves `/_ipx/…` via UnJS ipx); Cloudflare → `cloudflare`/`cloudflare_images`; Vercel/Netlify presets → theirs.
  - **Detect mode:** for external URLs (user content, CMS), use `getProviderForUrl` to keep the original CDN; fall back to the configured provider (proxy through ipx) or pass-through.
  - `ImageMetadata.layerUnpic`:
    - pixels: `@unpic/pixels` `getPixels` (PNG/JPEG); other formats via ipx/sharp adapter
    - color: `@unpic/placeholder` `getDominantColor`/`getPalette`
    - BlurHash: encode with `blurhash` [V dep of placeholder; encoder availability U]
- **Runs as a job** on upload (`Jobs` / PersistedQueue), writing metadata into the file field record (`{ width, height, dominantColor, blurhash }`). Traced spans `image.analyze`.
- **Security:**
  - **SSRF:** `getPixels(url)` fetches remote URLs, so only analyze pocket-stored files, or allowlisted hosts via HttpClient.
  - **ipx:** limit allowed sizes/formats to prevent resize-amplification abuse (allowlist breakpoints = `DEFAULT_RESOLUTIONS` subset).
  - **Protected files:** unpic URLs must carry pocket's file tokens (provider options / query passthrough) [U: per-provider query preservation].

### 4.2 Collections & API
- **File field option:** `Field.file({ image: { analyze: true, placeholder: "blurhash" | "dominantColor" | "none" } })`.
- **Records API:** optionally returns `imageMeta` alongside file names, plus a `/api/files/:collection/:id/:file` URL. Clients derive responsive URLs locally via unpic (no round-trip).
- **Schema:** `ImageMeta = Schema.Struct({ width: Int, height: Int, format: Literals, dominantColor: String, blurhash: optionalKey(String) })` so the client decodes it safely.

### 4.3 Foldkit (admin & Foldkit apps)
unpic's React/Vue/Solid components don't apply, but **`@unpic/core` is framework-agnostic**: `transformBaseImageProps(props) → attributes` maps directly onto Foldkit's attribute builder.
```ts
// @pocket/foldkit-image (sketch)
export const image = <Message>(h: HtmlBuilder<Message>, props: UnpicProps & { meta?: ImageMeta; parts?: ImageParts }) => {
  const attrs = transformBaseImageProps({
    src: props.src, width: props.width ?? props.meta?.width, height: props.height ?? props.meta?.height,
    layout: props.layout ?? "constrained", priority: props.priority, cdn: props.cdn,
    background: props.meta?.blurhash ? blurhashToCssGradientString(props.meta.blurhash) : props.meta?.dominantColor,
  })
  return h.img(toFoldkitAttributes(attrs, props.parts?.img))   // src, srcset, sizes, style, loading, decoding, fetchpriority
}
```
- **Pure view helper, not a bundle:** no state, no effects (the "pure helper" kind in [foldkit-primitives.md](foldkit-primitives.md)). Works in vanilla Foldkit.
- **Parts/Slots:** `img` (and `picture`/`source` for art direction via `transformBaseSourceProps`); foldkit-plus Mixins style it without wrappers.
- **SSR:** fully deterministic from props, so `serializeHtml` output matches client hydration; no layout shift (style carries aspect ratio).
- **Optional Bundle for stateful cases:**
  - `ImageLoad` bundle (Model: `loaded | error`, a Mount listening to `load`/`error`) for fade-in or error fallback
  - `Lightbox`/`Gallery` bundles (keyboard, focus trap, current index) that use `image()` inside
  - Upload preview: `ImageUpload` bundle (file → object URL → progress → server metadata)
- **Admin uses:** record grid thumbnails (with the Table bundle's cell renderer), file field previews, gallery view for collections with image fields, OG image previews.

### 4.4 foldkit-plus
| Integration | How |
|---|---|
| **Mixins** | `foldkit-mixins-ui`-style adapter: `ImageSlots` (`img`, `picture`, `source`) so Styles (rounded, aspect, object-fit variants) and Behaviors (zoom-on-hover, lightbox open) attach to images without wrappers |
| **Remote** | `foldkit-remote` entities carry `ImageMeta` fields; placeholders render immediately while records are cached/optimistic; responsive URLs computed client-side from entity data |
| **Agent** | Surfaces may expose image metadata (dimensions, dominant color, alt text) as projection fields; agent tools like `set_alt_text` map to existing Messages |
| **Proposal upstream** | A small `foldkit-image` package (or `@foldkit/ui` Image) built on `@unpic/core`: a first-class, zero-JS responsive image for the Foldkit ecosystem. Also a candidate registry item for pocket |

### 4.5 React / other frameworks (pocket users)
- Registry blocks use `@unpic/react` (or vue/solid/svelte) directly with pocket's `ImageMeta` → `background` placeholder.
- `@pocket/react` hook: `useImageProps(record, field, props)` → unpic props with provider and file token configured from `PocketProvider`.
- Web component variant: `@unpic/webc` / `@unpic/lit` for framework-agnostic registry items.

### 4.6 Emails, PDFs, OG images
- **Emails** (emailcn/jsx-email): no srcset support in most clients. Use `Images.url(source, { width: 600*2, format: "jpeg" })` for fixed-width retina images; provider transforms keep emails light.
- **PDFs** (pdfcn) and **OG images** (ogimagecn/Satori): fetch sized variants through `Images.url` to reduce render time and memory.
- **Dominant color** from `ImageMetadata` feeds theme accents in generated OG cards and email headers.

### 4.7 Deployment presets mapping
| pocket preset | unpic provider | Transform backend |
|---|---|---|
| Single binary / Docker | `ipx` | UnJS ipx inside pocket (`/_ipx/...`), cached in Storage |
| Cloudflare (Alchemy) | `cloudflare` (Image Resizing) or `cloudflare_images` | CF edge |
| Vercel / Netlify (embedded mode) | `vercel` / `netlify` | platform image optimizer |
| Supabase storage (adapter) | `supabase` | Supabase image transforms |
| External CMS/CDN URLs | detected via `getProviderForUrl` | original CDN |

## 5. Plan & phasing
| Phase | Deliverable |
|---|---|
| Phase 1 (core pocket) | `Images` service + `layerUnpic` (ipx provider); file field `image.analyze`; `ImageMetadata` job (pixels + dominant color); Foldkit `image()` helper in admin thumbnails |
| Phase 2 | BlurHash placeholders; Parts/Slots + Mixins adapter; React hook + registry blocks; email/PDF/OG use of `Images.url` |
| Phase 3 | Cloudflare/Vercel/Netlify/Supabase provider presets tied to deploy targets; detect-mode for external URLs; `ImageLoad`/`Gallery` bundles; propose `foldkit-image` upstream |

## 6. Risks & open questions
- `@unpic/placeholder` 0.1.x and `@unpic/pixels` Node-only decoding (pngjs/jpeg-js): verify Bun/Deno/Workers; for Workers use CF Images metadata or ipx instead [U].
- BlurHash **encoding** entry point (placeholder renders BlurHash; encoding may need `blurhash`'s `encode` directly) [U].
- Provider query-param preservation for pocket's **protected file tokens** [U per provider].
- ipx abuse limits (allowed widths/formats, cache) must be enforced server-side.
- AVIF/WebP negotiation differs by provider (`format: "auto"` support varies) [U].
- Maintainer concentration (single author), mitigated by small surface, MIT, and our `Images` contract.
