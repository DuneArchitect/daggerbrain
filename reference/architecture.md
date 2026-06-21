# Architecture Overview

Daggerbrain is a set of digital tools for the Daggerheart TTRPG (characters, campaigns, encounters, homebrew, live play, stream overlays, a blog). This page explains how the pieces fit together; each subsystem has its own deep-dive linked at the bottom.

## The stack

| Layer | Technology | Notes |
|---|---|---|
| Framework | **SvelteKit** + **Svelte 5** (runes) | fullstack; SSR for content, client-reactive for app data |
| Backend / DB | **Convex** | reactive document database; source of truth for access |
| Auth | **Clerk** | sessions; issues a `convex`-template JWT to authenticate Convex |
| Billing | Stripe (intended) → Convex entitlements | feature gates read Convex, not the billing provider |
| Deploy target | **Cloudflare Workers** | via `adapter-cloudflare`; 50ms CPU budget |
| Object storage | **Cloudflare R2** | app art + user uploads |
| Styling | **Tailwind v4** + bits-ui | CSS-variable theming |
| Observability | **Sentry** | server + client + source maps |

## The defining pattern: reactive, client-driven data

There is **no REST API for application data**. The browser holds an authenticated Convex client and subscribes directly to queries; the server (the Worker) mostly serves the SvelteKit shell, content pages, and image proxies.

```
Browser (Svelte 5 runes)                     Cloudflare Worker            Convex
─────────────────────────                    ─────────────────            ──────
state store (.svelte.ts)
  ├─ useQuery(get) ───────── reactive subscription ─────────────────────▶ query  ──▶ DB
  │     ◀───────────────────  live push on change  ─────────────────────────────────┘
  ├─ local $state copy (instant edits)
  └─ $effect debounce 200ms ─ mutation(update) ──────────────────────────▶ mutation ─▶ DB
                                                                            (permission
                                                                             check here)
SvelteKit pages ──── SSR / load ───▶ Worker ──▶ R2 (images), .svx content
Clerk session ─── getToken({template:'convex'}) ──▶ convexClient.setAuth
```

Three consequences worth internalizing:

1. **The client is optimistic and local-first.** Each editable entity has a local mutable copy; edits feel instant and sync on a debounce. A stable-snapshot diff reconciles server pushes against unsaved local edits to avoid loops. (See [Frontend State → the core sync pattern](./frontend-state.md#the-core-sync-pattern).)
2. **Permissions live only in Convex.** The UI hides controls, but every read/write re-checks ownership/membership server-side. Never trust client-supplied ids. (See [Convex Backend → Permissions](./convex-backend.md#permissions-permissionsts).)
3. **The Worker stays thin.** With a 50ms CPU budget, real work happens in Convex and the browser; the Worker handles SSR, content, and R2 streaming.

## Domain model in one breath

Everything orbits the **compendium** — 15 game-content item types (weapons, armor, classes, subclasses, domains, domain cards, ancestry/community/transformation cards, beastforms, loot, consumables, adversaries, environments). These 15 types recur at every layer: Zod schema → Convex table → homebrew form → preview component.

- A **character** stores choices (class, level-up picks, equipment, marks) plus a resolved **compendium scope** (official sources + homebrew + campaign content). A large pure function, `derive_character.ts`, turns choices + compendium into the playable sheet.
- A **campaign** is one document embedding its members and character roster; it shares a fear track, countdowns, notes, a homebrew vault, and dice history with all members in real time, and can expose a public **stream overlay** by token.
- An **encounter** is a GM's adversary/environment plan with a battle-points difficulty budget.

## Source tree map

```
src/
├── convex/            # backend: schema, schemas/, functions/, permissions, http, constants
├── compendium/SRD/    # bundled official game data (15 categories)
├── routes/            # SvelteKit pages: (app) group, stream/, text endpoints
├── lib/
│   ├── state/         # Svelte 5 reactive stores (one per domain) + derive_character
│   ├── components/    # ui/ primitives + feature components (character-sheet, campaigns, homebrew, …)
│   ├── compendium/    # official-sources registry/merge
│   ├── remote/        # SvelteKit remote functions (R2 upload)
│   ├── server/        # blog post + article loading
│   ├── articles/      # blog rendering components
│   ├── pdf/           # character sheet PDF export
│   ├── constants/ schemas/ hooks/ assets/
│   └── utils.ts       # cn() + game-mechanics helpers
├── posts/             # .svx blog content
├── hooks.server.ts    # Sentry + Clerk + maintenance sequence
├── hooks.client.ts    # client Sentry
└── service-worker.ts
```

## Where to go next

| To understand… | Read |
|---|---|
| Tables, functions, permissions, entitlements, auth | [Convex Backend](./convex-backend.md) |
| Routes, route groups, auth wiring, server hooks | [Routing & Pages](./routing-and-pages.md) |
| The reactive stores and the sync mechanism | [Frontend State](./frontend-state.md) |
| Game data, sources, content layering | [Compendium & SRD](./compendium-and-srd.md) |
| Sheet, editor, leveling, the derivation engine | [Character Tools](./character-tools.md) |
| Campaigns, live play, dice, stream, encounters | [Campaigns & Live Play](./campaigns-and-live-play.md) |
| Authoring custom content | [Homebrew](./homebrew.md) |
| Components, primitives, theming, SEO | [UI & Theming](./ui-and-theming.md) |
| Build, Cloudflare, R2, Sentry, blog | [Infrastructure & Content](./infrastructure-and-content.md) |
