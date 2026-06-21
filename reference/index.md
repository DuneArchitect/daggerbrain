# Daggerbrain Reference

A structural reference for the Daggerbrain codebase — a DNDBeyond-style suite of digital tools for the **Daggerheart** TTRPG (character builder, campaign manager, encounter builder, homebrew authoring, live-play dashboards, stream overlays, and a markdown blog).

This folder is **architecture-level documentation**: what the major pieces are, where they live, and how they fit together. It complements the design-intent docs in [`../docs/`](../docs/) (billing, permissions, campaigns) and the setup instructions in [`../README.md`](../README.md) / [`../CLAUDE.md`](../CLAUDE.md).

> Generated from a deep scan of the codebase. If code and docs disagree, code wins — flag the drift.

## Stack at a glance

**SvelteKit + Svelte 5** frontend · **Convex** reactive backend/DB · **Clerk** auth · **Stripe→Convex** entitlements · **Cloudflare Workers** deploy · **Cloudflare R2** images · **Tailwind v4** + bits-ui · **Sentry** observability.

The defining trait: the browser subscribes to Convex **directly** over reactive queries (no REST API for app data), keeps a local-first editable copy, and syncs back on a debounce. Permissions are enforced only in Convex. Start with [Architecture Overview](./architecture.md) for how this all flows.

## Documents

| # | Document | What's inside |
|---|---|---|
| 1 | [Architecture Overview](./architecture.md) | the stack, the reactive data-flow pattern, source-tree map, where to go next |
| 2 | [Convex Backend](./convex-backend.md) | schema & tables, the Zod→Convex pattern, functions, permissions, entitlements, HTTP webhook, auth |
| 3 | [Routing & Pages](./routing-and-pages.md) | the route tree, `(app)`/`(edit)` groups, Clerk↔Convex auth wiring, server hooks, special endpoints |
| 4 | [Frontend State](./frontend-state.md) | the Svelte 5 runes stores, the local-first sync pattern, the store-by-store breakdown |
| 5 | [Compendium & SRD](./compendium-and-srd.md) | the 15 game-content item types, bundled SRD data, sources, content layering |
| 6 | [Character Tools](./character-tools.md) | the sheet, the editor & leveling system, the `derive_character` engine, PDF export |
| 7 | [Campaigns & Live Play](./campaigns-and-live-play.md) | campaign model, live dashboards, the dice engine, stream overlays, encounters |
| 8 | [Homebrew](./homebrew.md) | the uniform per-type form/normalize/errors pattern, shared fragments, vault flow |
| 9 | [UI & Theming](./ui-and-theming.md) | bits-ui primitives, `utils.ts`, the CSS-variable theme system, SEO |
| 10 | [Infrastructure & Content](./infrastructure-and-content.md) | build, Cloudflare runtime, R2 image system, service worker, Sentry, the mdsvex blog |

## Reading paths

- **New to the codebase?** [Architecture](./architecture.md) → [Convex Backend](./convex-backend.md) → [Frontend State](./frontend-state.md).
- **Working on the character builder?** [Compendium & SRD](./compendium-and-srd.md) → [Character Tools](./character-tools.md) → [Frontend State](./frontend-state.md).
- **Working on multiplayer / live features?** [Campaigns & Live Play](./campaigns-and-live-play.md) → [Convex Backend](./convex-backend.md).
- **Adding game content or homebrew types?** [Compendium & SRD](./compendium-and-srd.md) → [Homebrew](./homebrew.md).
- **Platform / deploy / content?** [Infrastructure & Content](./infrastructure-and-content.md) → [Routing & Pages](./routing-and-pages.md).

## Key conventions to know

- **One Zod schema per type, reused everywhere** — `zodToConvex` makes it a DB validator; the same schema validates homebrew forms client-side.
- **The 15 compendium item types recur at every layer** — schema, table, homebrew form triplet, preview component. Learn one, you know all fifteen.
- **State is local-first with debounced sync** — edit the local `$state` copy; a 200ms-debounced mutation persists it; a stable-snapshot diff reconciles server pushes.
- **`owner_clerk_id` is the current user key** — the Clerk subject id; the longer-term move to internal Convex ids is noted in [`../docs/permissions.md`](../docs/permissions.md).
- **The Worker is thin (50ms CPU)** — heavy logic lives in Convex and the browser.
