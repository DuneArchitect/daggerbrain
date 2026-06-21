# Routing, Pages & Auth Wiring

The frontend is [SvelteKit](https://svelte.dev/docs/kit) running on Svelte 5 (runes). Routes live in `src/routes/`. There is **no traditional REST API for app data** — pages talk to Convex directly over reactive subscriptions, so most page/layout files have no `+page.server.ts`. Server load files exist only for content (blog/changelog) and a few text endpoints.

## Route tree

```
src/routes/
├── +layout.svelte              # root: dark mode, fonts, head
├── +layout.server.ts           # buildClerkProps(locals.auth()) -> client
├── +error.svelte               # error page
├── layout.css                  # global Tailwind layer + CSS variables
│
├── (app)/                      # route group (no URL prefix) — the main app
│   ├── +layout.svelte          # setupConvex(), <ClerkProvider>, <AppShell>
│   ├── +page.svelte            # landing page
│   ├── app-shell.svelte        # <Seo> + <Navbar> + <Toaster> + page content
│   │
│   ├── characters/             # list -> [id] sheet -> (edit) editor group
│   │   ├── +page.svelte
│   │   └── [id]/
│   │       ├── +page.svelte                # character SHEET
│   │       └── (edit)/                     # editor group (no URL prefix)
│   │           ├── edit | class | heritage | traits | experiences | equipment
│   ├── campaigns/
│   │   ├── +page.svelte                    # list
│   │   ├── [id]/+page.svelte               # campaign dashboard
│   │   ├── [id]/live/+page.svelte          # live session (GM/Player)
│   │   └── join/[uid]/+page.svelte         # invite-join flow
│   ├── encounters/             # list -> [id] builder
│   ├── homebrew/               # list -> [type]/[uid] editor (dynamic dispatch)
│   ├── posts/                  # blog list -> [slug] (server-loaded .svx)
│   ├── changelog/              # single .svx page
│   ├── profile/                # account + billing status
│   ├── faq | contact | roadmap | privacy | terms
│   └── api/
│       ├── images/[...key]/+server.ts            # proxy R2_IMAGES
│       └── usercontent/images/[...key]/+server.ts # proxy R2_USERCONTENT
│
├── maintenance/+page.svelte    # shown when MAINTENANCE_MODE redirect fires
├── stream/                     # PUBLIC, unauthenticated overlay
│   ├── +layout.svelte          # own setupConvex(), noindex
│   └── [token]/+page.svelte    # token-addressed OBS overlay
├── llms.txt/+server.ts         # AI discovery text
├── robots.txt/+server.ts
└── sitemap.xml/+server.ts      # static pages + published posts
```

### Route-group conventions

- **`(app)`** wraps the authenticated app without adding a URL segment — `/characters`, not `/app/characters`. It is where Convex and Clerk are initialized.
- **`(edit)`** groups the six character-editor steps under `characters/[id]/` so they share one editor layout (tabs, prev/next, portrait upload) while resolving as `/characters/[id]/class`, `/characters/[id]/traits`, etc.

### Dynamic params

| Param | Used in | Meaning |
|---|---|---|
| `[id]` | characters, campaigns, encounters | Convex document id |
| `[slug]` | posts | `.svx` filename / frontmatter slug |
| `[uid]` | campaigns/join, homebrew | invite code / homebrew doc id |
| `[type]` | homebrew | homebrew item type (dispatches to the right form) |
| `[token]` | stream | opaque overlay token (the auth) |
| `[...key]` | api/images | catch-all reconstructing an R2 object key |

## Auth wiring (Clerk → Convex)

Authentication is **Clerk**; the Convex client is authenticated with a Clerk-issued JWT minted from a template named `convex`.

```
hooks.server.ts  → withClerkHandler()              # parses session, sets locals.auth()
+layout.server.ts → buildClerkProps(locals.auth()) # ships auth state to the client
(app)/+layout.svelte → <ClerkProvider> + setupConvex(PUBLIC_CONVEX_URL)
user.svelte.ts → convexClient.setAuth(() => session.getToken({ template: 'convex' }))
```

So the chain is: **Clerk session → `getToken({ template: 'convex' })` → `convexClient.setAuth` → every Convex query/mutation is authenticated as that user.** See [Frontend State](./frontend-state.md) for the bridge code, and [Convex Backend](./convex-backend.md#authentication) for the server side.

There are **no explicit route guards**. Protection is implicit: Convex functions reject unauthenticated callers, and feature pages use Clerk's `<SignedIn>/<SignedOut>` components (e.g. homebrew shows marketing copy when signed out).

## Server hooks (`src/hooks.server.ts`)

Handlers run as a `sequence` (order matters — Clerk must run before maintenance so `locals.auth()` is populated):

1. `Sentry.initCloudflareSentryHandle` — Sentry for the Workers runtime
2. `Sentry.sentryHandle()` — request tracing/error capture
3. `withClerkHandler()` — Clerk session parsing
4. `maintenanceModeHandle` — if `MAINTENANCE_MODE === 'true'`, redirect everyone except `ADMIN_CLERK_ID`s to `/maintenance`

`handleError` is wrapped with `Sentry.handleErrorWithSentry()`. Client-side Sentry (with session replay, text masked) initializes in `src/hooks.client.ts`.

## Data-loading patterns

- **App data (characters, campaigns, encounters, homebrew):** no server load — loaded client-side via Convex `useQuery`/`onUpdate` through the state contexts. Pages set a context's `id` and reactive data flows in.
- **Content (posts, changelog):** `+page.server.ts` uses `import.meta.glob` over `src/posts/*.svx` (eager metadata + `?raw` source) and renders via the article helpers. See [Infrastructure & Content](./infrastructure-and-content.md).
- **Root layout:** `+layout.server.ts` only forwards Clerk props.

## Special endpoints

| Route | Output |
|---|---|
| `llms.txt` | plain-text list of public pages + note that user content is private |
| `robots.txt` | allow-all + sitemap link |
| `sitemap.xml` | static public pages (from SEO route definitions) + published posts |
| `stream/[token]` | the only **public, Clerk-less** app surface; reads overlay state by token |

## App shell

`(app)/app-shell.svelte` wraps every authenticated page with `<Seo>` (per-route meta + JSON-LD, see [UI & Theming](./ui-and-theming.md#seo)), the `<Navbar>` (Play / Community dropdowns, Clerk sign-in/up or user button), and a `<Toaster>` for notifications.

## Cross-references

- Reactive state contexts the pages consume → [Frontend State](./frontend-state.md)
- Character sheet & editor pages → [Character Tools](./character-tools.md)
- Campaign / live / stream pages → [Campaigns & Live Play](./campaigns-and-live-play.md)
- Image proxy routes, service worker, blog loading → [Infrastructure & Content](./infrastructure-and-content.md)
