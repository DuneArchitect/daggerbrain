# UI Components & Theming

The UI is **Tailwind CSS v4** + a set of **bits-ui** primitive wrappers (a shadcn-svelte-style port), with a CSS-variable-driven theme system tuned for the Daggerheart aesthetic.

## UI primitives (`src/lib/components/ui/`)

These are thin, styled wrappers over [bits-ui](https://bits-ui.com) headless primitives. Each component directory exposes an `index.ts` that re-exports the parts as both `Root` and PascalCase aliases (`Root as Dialog`, `Content as DialogContent`, …), the convention shadcn-svelte popularized.

| Directory | Purpose |
|---|---|
| `button`, `button-group` | buttons (variant/size via `tailwind-variants`), grouped sets |
| `dialog`, `sheet`, `popover` | modal, slide-out panel, floating panel |
| `command` | searchable command palette / combobox menu |
| `select`, `checkbox`, `switch`, `input`, `textarea`, `label` | form controls |
| `tabs`, `collapsible`, `separator`, `scroll-area` | layout/disclosure |
| `skeleton` | loading placeholder |
| `sonner` | toast notifications (`Toaster`) |

Styling uses **`tailwind-variants` (`tv()`)** for typed variant systems (see `button/button.svelte`) and the `cn()` helper everywhere else.

## `src/lib/utils.ts`

Beyond `cn()` (= `twMerge(clsx(...))`), this file is also the home of **game-mechanics helpers** reused across the sheet, catalogs, and homebrew:

- `cn`, `capitalize`, `formatDate`, `formatTimeAgo`
- `renderMarkdown` — `marked` → DOMPurify-sanitized HTML
- `level_to_tier` / `tier_to_min_level`, `increaseDie`, `increase_range`, `applyProficiencyToDice`, `addBonusDamageDie`, `parseDiceString`
- `merge_compendium_content` — the shallow record-merge behind [content layering](./compendium-and-srd.md#sources-and-how-content-layers-combine)
- Type helpers: `WithoutChild`, `WithoutChildren`, `WithElementRef`

## Theming

The visual theme is entirely **CSS variables**, defined in `src/routes/layout.css` and exposed to Tailwind v4 via `@theme inline`.

- **Global tokens** (`:root` / `.dark`): `--background`, `--foreground`, `--card`, `--primary` (+`--primary-muted`), `--accent`, `--destructive`, `--muted`, `--border`, `--ring`, plus game-specific `--hope` (gold), `--fear` (purple), and dice colors. The app runs dark-mode-first (`mode-watcher`).
- **Named palettes** (`src/lib/constants/themes.ts`): a `THEMES` map of ~10 palettes named after the domains (default, arcana, blade, bone, codex, grace, midnight, sage, splendor, valor), each a full set of color tokens. Validated by `ThemeSchema` (`src/lib/schemas/themes.ts`, re-exported into Convex via `src/convex/schemas/themes.ts`).
- **Character sheet backgrounds** (`CHARACTER_SHEET_BACKGROUNDS`): selectable art backgrounds (mountains, forge, stormy, rocky, maps), each an enhanced-img + preview url.
- **Fonts:** Inter (variable, body) and Eveleth Clean (display, headings) via `@font-face`.
- **Custom utilities:** `.card-shadow`, `.card-shadow-lg`, `.inset-shadow`. The Clerk auth widget is themed through `--clerk-*` variables.

Tailwind v4 is wired in `vite.config.ts` via `@tailwindcss/vite`; `@sveltejs/enhanced-img` optimizes images and Prettier sorts classes against `layout.css`.

## Decorations (`src/lib/components/decorations/`)

Game-flavored visuals: `class-banner` / `domain-banner` (75×120 banners with dual domain icons, gradients, and clip-path edges), `domain-icon` (raster or CSS-masked local SVG), badges (`beta`, `campaign`, `homebrew`, `maintenance-banner`), `device-mockup`, `subscribe-button`.

## Utility components (`src/lib/components/utility/`)

`card-carousel` (snap-scroll with keyboard nav + persisted position), `dropdown`, `choice-select`, `experience-select`, `loader` (delayed, reduced-motion-aware spinner), `load-error`, and `user-image-uploader` (drives the [R2 remote upload](./infrastructure-and-content.md#r2-image-system)). `shared/safe-delete.svelte` is the type-to-confirm destructive-action widget.

## SEO

`src/lib/components/seo/seo.ts` + `seo.svelte` centralize metadata. `seo.ts` holds site constants and **route definitions** (public vs. private/noindex), and builds per-route `PageSeo` and **schema.org JSON-LD** (`WebSite`, `Organization`, `SoftwareApplication`, FAQ, etc.). `seo.svelte` (rendered once in the [app shell](./routing-and-pages.md#app-shell)) emits `<title>`, description, robots, canonical, Open Graph, Twitter Card, and the JSON-LD script — using `page.data.seo` when a route provides it, else deriving from the route id. The same route definitions feed `sitemap.xml` and `llms.txt`.

## Cross-references

- Where these components are mounted → [Routing & Pages](./routing-and-pages.md)
- Image optimization, fonts, and the build → [Infrastructure & Content](./infrastructure-and-content.md)
- Domain colors used by banners/icons → [Compendium & SRD](./compendium-and-srd.md#domains-classes-subclasses-relationships)
