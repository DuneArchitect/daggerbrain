# Frontend State

State lives in `src/lib/state/*.svelte.ts` and is built on **Svelte 5 runes** (`$state`, `$derived`, `$derived.by`, `$effect`). Each domain has one store, exposed as a **context singleton** (`setXContext()` / `getXContext()`) so pages set it once and descendant components read it without prop drilling.

The stores are the bridge between the UI and [Convex](./convex-backend.md): they hold a local editable copy, subscribe to server data reactively, and write changes back with a debounce.

## The core sync pattern

Every editable store (character, campaign, encounter) follows the same shape:

1. **Subscribe** to the server document with `useQuery(api.functions.X.get, () => id ? { id } : 'skip')`.
2. Keep a **local mutable copy** in `$state` that the UI edits directly.
3. **Server → local:** an `$effect` watches the server snapshot and applies it *unless* there are unsaved local edits.
4. **Local → server:** an `$effect` watches the local snapshot and fires a `convexClient.mutation(...update)` after a **200ms debounce**.
5. **Loop prevention:** both sides are compared via `stableSnapshot()` — `JSON.stringify` with **sorted keys** — and an `appliedServerSnapshot` marker distinguishes "new server change" from "my own echo".

```ts
function stableSnapshot(value: unknown): string {
  return JSON.stringify(value, (_, v) =>
    !v || typeof v !== 'object' || Array.isArray(v)
      ? v
      : Object.fromEntries(Object.entries(v).sort(([a], [b]) => a.localeCompare(b))));
}
```

This is why editing a character feels instant but still syncs reliably across clients.

## The stores

### `user.svelte.ts` — identity, entitlements, the auth bridge
- Installs the Clerk→Convex token: `convexClient.setAuth(() => session.getToken({ template: 'convex' }))`.
- Queries `users.get` and `entitlements.getFeatures`; **auto-creates** the user record (`users.create`) on first load.
- Exposes `user`, `features`, and derived limits `character_limits` / `homebrew_limits` (free caps vs. `unlimited_*` slugs — see [Convex Entitlements](./convex-backend.md#entitlements-paid-access)).
- Helpers: `createCharacter()`, `deleteCharacter()`, `uploadImage()`.

### `character.svelte.ts` — the central store
Holds `id`, the local `character`, and the derived view. It assembles the character's **compendium scope** (official sources + homebrew vault + campaign vault, see [Compendium](./compendium-and-srd.md)) and feeds `(character, compendium)` into the derivation engine:

```ts
const character_derivation = $derived.by(() =>
  character && ready_character_compendium
    ? derive_character_state(character, ready_character_compendium)
    : undefined);
```

Exposes `derived_character_data`, `character_compendium`, `available_source_keys`, `canEdit`/`isOwner`, plus inventory helpers (`addToInventory`, `equipItem`, `addScar`, …). The editor mutates `character` directly; the sheet reads `derived_character_data`.

### `derive_character.ts` — the derivation engine
A large **pure function** (`derive_character_data` / `derive_character_state`) that turns a stored character + compendium into a fully-resolved sheet. It is the single most important piece of domain logic in the app — documented in depth in [Character Tools → Derivation](./character-tools.md#the-derivation-engine).

### `campaign.svelte.ts` — campaign + roster + vault + dice
- Queries `campaigns.get` and `campaigns.getDiceHistory`.
- **Manually** subscribes (via `convexClient.onUpdate`) to every roster character and every vault homebrew item, tracking unsub functions in a plain `Map` and results/errors in reactive `SvelteMap`s. An `$effect` adds/removes subscriptions as the roster/vault changes; `onDestroy` cleans them all up.
- Exposes `isGm`, `roster` (active/inactive), `members`, `compendium` (vault), `diceHistory`, `inviteCode`, plus `addToVault`/`removeFromVault`/`addRollToHistory`.

### `dice.svelte.ts` — 3D dice engine
Wraps `@3d-dice/dice-box`. Exposes `roll()`, `rerollDie()`, `history`, `lastRoll`, `isRolling`, and **callback registries** (`onRollEnd`, `onRollStart`, `onPickerOpen`) used to broadcast rolls into a campaign without coupling the two stores. Full mechanics (Hope/Fear duality, reroll merging) in [Campaigns & Live Play → Dice](./campaigns-and-live-play.md#dice-system).

### `encounters.svelte.ts` — encounter + battle points
Standard sync pattern plus a `derived_battle_points` computation (budget vs. spent, difficulty band). See [Campaigns & Live Play → Encounters](./campaigns-and-live-play.md#encounters).

### `homebrew.svelte.ts` — the user's homebrew vault
Subscribes to each item in the user's `homebrew_vault` and assembles them into a `CompendiumContent`. Helpers `addItem`/`updateItem`/`removeItem` wrap the `homebrew.*` mutations. See [Homebrew](./homebrew.md).

### `sources.svelte.ts` — official sources
Queries `sources.list` and builds a `CompendiumContent` from the user's enabled official source keys via `getOfficialCompendiumFromSourceKeys`.

### `compendium-vault.svelte.ts` — reusable subscription factory
`createVaultCompendiumSubscription({ convexClient, getVault, sourceKeyOverride })` is the shared machinery behind the homebrew **and** campaign vaults: it subscribes to a set of item ids, rebuilds the `CompendiumContent` as results arrive, and can stamp a `source_key` override (`'Homebrew'` / `'Campaign'`).

### `localstorage.svelte.ts` — UI preferences
A Zod-validated `app_preferences` object persisted to `localStorage` (debounced). Stores per-entity UI state keyed by id: selected feature tab, carousel position, dashboard panel toggles, preview visibility, etc. Purely client-side; never touches Convex.

## Hooks (`src/lib/hooks/`)

- `is-mobile.svelte.ts` — `IsMobile` extends Svelte's `MediaQuery` (`max-width: 767px`); read `.current`.
- `use-clipboard.svelte.ts` — `UseClipboard` copy helper with a transient `copied`/`status` flag.

## Runes cheat-sheet (as used here)

| Rune | Role in this codebase |
|---|---|
| `$state` | local editable entity, loading flags |
| `$derived` | thin projections of one reactive value |
| `$derived.by` | expensive computed values (character derivation, roster, vault merge) |
| `$effect` | Convex subscriptions, debounced mutations, server↔local sync; returns cleanup |
| `SvelteMap` | reactive per-item subscription results (campaign roster/vault, homebrew) |

## Cross-references

- Server functions these stores call → [Convex Backend](./convex-backend.md)
- The derivation engine in detail → [Character Tools](./character-tools.md)
- How content layers merge into a scope → [Compendium & SRD](./compendium-and-srd.md)
