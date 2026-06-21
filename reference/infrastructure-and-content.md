# Infrastructure & Content

This covers everything outside the feature domains: the build, the Cloudflare runtime, image storage, observability, and the markdown blog.

## Build & deploy

| File | Role |
|---|---|
| `vite.config.ts` | plugin chain: `sentrySvelteKit()` → `tailwindcss()` → `enhancedImages()` → `sveltekit()` |
| `svelte.config.js` | `adapter-cloudflare`; preprocessors `[mdsvex(), vitePreprocess()]`; `.svelte`/`.svx` extensions; `@convex` alias → `./src/convex`; experimental `remoteFunctions`, server tracing/instrumentation; `compilerOptions.experimental.async` |
| `wrangler.jsonc` | Worker `daggerbook`; entry `.svelte-kit/cloudflare/_worker.js`; `nodejs_compat`; `cpu_ms: 50`; `ASSETS` binding; R2 bindings `R2_IMAGES`, `R2_USERCONTENT` |
| `convex.json` | Convex functions root `src/convex/`, `tsgo` compiler |
| `tsconfig.json` / `src/convex/tsconfig.json` | app vs. backend TS config |

**Flow:** `npm run build` (Vite + adapter-cloudflare) emits `.svelte-kit/cloudflare/_worker.js`; `wrangler deploy` ships the worker + static assets. Convex is deployed separately via `npx convex`. See the [README](../README.md) and [CLAUDE.md](../CLAUDE.md) for local dev (`npm run convex:dev` + `npm run dev`).

## Cloudflare runtime

The app runs on **Cloudflare Workers**. Platform bindings are typed in `src/app.d.ts` (`App.Platform.env`) and the generated `src/worker-configuration.d.ts`:

- `R2_IMAGES`, `R2_USERCONTENT` — R2 (S3-compatible) object storage.
- `ASSETS` — static asset fetcher.
- Secrets (Clerk, Convex, Sentry, R2) arrive as env.

A tight **50ms CPU budget** per request shapes the architecture: heavy data work is pushed to Convex and the client, not the Worker. The plain Vite dev server has no R2 bindings, so image routes degrade gracefully (503 `dependency_unavailable`) unless you run `wrangler dev`.

## R2 image system

Two halves: serving and uploading.

**Serving (proxy routes).** `api/images/[...key]` and `api/usercontent/images/[...key]` are catch-all `GET` handlers that join the path back into an R2 key, `r2.get(key)`, and stream the object with `Cache-Control: public, max-age=31536000, immutable`. Compendium/app art comes from `R2_IMAGES`; user uploads from `R2_USERCONTENT`. User URLs embed `{userId}/{uuid}`, making them effectively unguessable (the sharing model relies on that rather than per-request auth).

**Uploading (SvelteKit remote functions).** `src/lib/remote/images.remote.ts` uses SvelteKit's experimental **remote functions** (`*.remote.ts`, enabled in `svelte.config.js`) — server-side handlers the client imports and calls like local functions, no manual endpoint. `upload_user_image` validates size (≤5MB) and MIME type, requires auth, writes to `R2_USERCONTENT` at `{userId}/{uuid}.{ext}`, and returns the public proxy URL. `src/lib/remote/utils.ts` provides the `AppResult` success/failure union and the `require_auth` / `require_r2_*` guards. The UI entry point is `utility/user-image-uploader.svelte`.

## Service worker

`src/service-worker.ts` precaches the SvelteKit build manifest (`[...build, ...files]`) under a `cache-${version}` key, deletes stale caches on activate, and serves cached same-origin GETs for precached paths (network fallback otherwise) — basic offline/perf support that invalidates cleanly on each deploy.

## Observability (Sentry)

- **Server** (`src/hooks.server.ts`): `initCloudflareSentryHandle` + `sentryHandle()` lead the [handle sequence](./routing-and-pages.md#server-hooks-srchooksserverts); errors flow through `handleErrorWithSentry`.
- **Client** (`src/hooks.client.ts`): `Sentry.init` with 100% traces, session replay (10% baseline / 100% on error, all text masked, `sendDefaultPii: false`).
- **Build:** `sentrySvelteKit({ org, project })` in `vite.config.ts` handles source-map upload. DSNs/org/project are placeholders to fill in per the [CLAUDE.md](../CLAUDE.md) Sentry note.

## Content & blog (mdsvex)

Markdown posts are authored as `.svx` files (mdsvex: YAML frontmatter + Markdown, compiled to a Svelte component exporting `metadata`).

- **Posts:** live in `src/posts/*.svx`. `src/lib/server/posts.ts` loads them with `import.meta.glob` (eager metadata + lazy content), validates frontmatter, filters `published`, sorts by date, and paginates. Routes: `posts/+page.server.ts` (list) and `posts/[slug]/+page.server.ts` + `+page.ts` (detail, 404 if missing/unpublished).
- **Article rendering:** `src/lib/server/article.ts` extracts headings (for the TOC, with deduped slug anchors) and builds article SEO + `BlogPosting` JSON-LD. UI in `src/lib/articles/`: `article-header` (title/author/date), `article-toc` (sticky desktop / dropdown mobile, scroll-spy), `article-content` (styled `<article>` body).
- **Changelog:** `changelog/changelog.svx` reuses the same article machinery via `loadArticlePageData`.
- **Frontmatter shape** (typed in `src/app.d.ts`): `title`, `description`, `date`, `published`, `author { name, role?, avatar? }`, optional `updated`, `coverImage`.

## Cross-references

- Routes, hooks order, and auth wiring → [Routing & Pages](./routing-and-pages.md)
- The Convex backend deployed alongside the Worker → [Convex Backend](./convex-backend.md)
- The image uploader and SEO components → [UI & Theming](./ui-and-theming.md)
