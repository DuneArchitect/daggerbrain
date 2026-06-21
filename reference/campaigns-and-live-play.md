# Campaigns, Live Play, Dice & Encounters

These features turn the app from a character builder into a table. They share one trait: **real-time multi-client sync** via Convex reactive queries. A GM changes the fear track and every player's screen — and the stream overlay — updates.

## Campaign data model

A campaign is a **single Convex document** (`campaigns` table) with embedded arrays — not normalized join tables:

```
campaigns: {
  invite_code,
  campaign: { name, fear_track, countdowns[], homebrew_vault,
              fear_visible_to_players?, public_notes?, private_notes?,
              current_encounter_id? },
  members:    [{ clerk_id, display_name, role: 'GM' | 'Player' }],
  characters: [{ character_id, status: 'active' | 'unclaimed', claimed_by_clerk_id? }]
}
```

> Note: `docs/campaigns-doc.md` describes an *intended* normalized model (`campaign_membership`, `campaign_characters` tables). The **shipped** implementation embeds `members`/`characters` in the campaign document — treat `schema.ts` and `functions/campaigns.ts` as authoritative. The role semantics in that doc still hold.

Permissions and the function list are in [Convex Backend](./convex-backend.md); the reactive store is `campaign.svelte.ts` in [Frontend State](./frontend-state.md).

## Campaign page (`campaigns/[id]`)

The layout seeds three contexts (campaign, dice, encounters). A single `isGm` flag (derived from membership role) gates the UI:

- **Shared / GM-editable:** characters roster, public notes, invite link, fear track, countdowns.
- **GM-only:** campaign vault (homebrew sharing), private notes, stream settings, fear-visibility toggle.
- **Player-only:** claim an unclaimed character, leave campaign, set display name.

Key components (`components/campaigns/`): `campaign-header`, `campaign-characters` (roster with claim/add/remove), `campaign-vault` (+ `homebrew-items-combobox`, see [Homebrew](./homebrew.md#campaign-vault-integration)), `campaign-public-notes` / `campaign-private-notes` (markdown), `campaign-invite-link` (12-char code → `/campaigns/join/[uid]`), `character-preview`, `stream-settings-dialog`.

## Live session (`campaigns/[id]/live`)

One page renders either `campaign-live-dashboard-gm.svelte` or `…-player.svelte` (panel toggles persisted to localStorage; mobile uses tabs). Real-time sync is just the standard Convex pattern: `campaigns.get` + `getDiceHistory` push updates; local edits debounce back at 200ms.

- **GM dashboard:** roster with live HP/Armor/Hope, encounter loader + battle points, fear editor, countdown manager, dice roller + log, notes/vault.
- **Player dashboard:** active characters, fear track *only if* `fear_visible_to_players`, countdowns *only if* per-countdown `visibleToPlayers`, read-only public notes, personal dice roller.

Live components live in `components/campaigns/live/`: `campaign-live-characters`, `campaign-live-encounter`, `campaign-live-fear`, `campaign-live-notes`, `countdown-sheet`.

## Dice system

State: `dice.svelte.ts`; UI in `components/dice/`; 3D rendering via `@3d-dice/dice-box` (types in `src/types/3d-dice-dice-box.d.ts`, assets under `static/dice-box/`).

### Dice types
Standard `d4 d6 d8 d10 d12 d20` plus Daggerheart's `hope` (gold d12), `fear` (purple d12), `advantage` (green d6), `disadvantage` (red d6). Schema in `src/convex/schemas/dice.ts`.

### Roll pipeline
1. `dice-picker.svelte` builds a `RollInput` (dice + modifier + name).
2. `prepareRollObjects()` groups dice by type/color and records a `rollOrder` to map physics results back.
3. `DiceBox.roll()` runs the ~3s physics sim; `onRollComplete` → `finalizeRoll()` maps `DieResult[]` back onto the `Roll.dice`, appends to local history (cap 20), and fires `onRollEnd` callbacks. Dice fade off-screen after ~2.4s.

### Hope/Fear duality
`getDescription(roll)` reads the result: matching hope & fear values → **Critical Success**; more fear → **with Fear**; more hope → **with Hope**. `getTotal()` sums dice + modifier, adding advantage and subtracting disadvantage. `roll-summary.svelte` color-codes the outcome.

### Reroll & campaign broadcast
`rerollDie(sourceRoll, indices)` re-renders only the selected dice and merges results back in place (`isReroll: true`). In a campaign, an `onRollEnd` callback calls `campaign.addRollToHistory(roll)` → `campaigns.updateDiceHistory` (persisted per campaign, capped ~50, tagged with `rollerName`). `dice-log-sheet.svelte` shows the shared history.

## Stream overlay (`/stream/[token]`)

The only **public, unauthenticated** surface. A campaign generates an opaque ~40-char token; OBS/Streamlabs loads `/stream/[token]` as a browser source.

- Backend (`streamOverlays.ts`, `stream_overlays` table): `createOrRotate` (mint/rotate token), `getOverlayState(token)` (returns state only when `enabled`), `updateSettings`.
- The overlay reads `campaign.fear_track` and player-visible `countdowns`, rendering only the enabled `modules` (`fear`, `countdowns`). `settings` (show label, group-with-fear) and `layout` (per-module x / y / scale) position the elements; CSS `transform: scale()` with transparent background makes it capture cleanly.
- `stream-settings-dialog.svelte` (GM) toggles modules, edits position/scale, shows the overlay URL, and rotates the token. The page is `noindex`.

## Encounters

State: `encounters.svelte.ts`; UI in `components/encounters/`; schema in `src/convex/schemas/encounters.ts`.

An `Encounter` holds metadata (name, description, conditions, player count, tier, massive-damage/bonus-damage flags, extra battle points) and an `items[]` array of adversaries (each with `instances[]` tracking per-copy name/conditions/marked HP & stress) and environments.

`encounter.svelte` is the builder; adversaries/environments are added via `adversary-selector-sheet` / `environment-selector-sheet` (backed by the catalogs). `encounter-battle-points.svelte` shows the **difficulty budget**:

- **Budget** ≈ `players × 3 + 2 + extra + lower-tier count`, with adjustments (`+1` if no major threats; `−2` for 2+ Solos; `−2` if bonus damage).
- **Spent** = sum of per-type costs (Minion ~1/group, Social/Support 1, Horde/Ranged/Skulk/Standard 2, Leader 3, Bruiser 4, Solo 5).
- **Difficulty band:** Easy (spent < budget), Normal (=), Hard (+1), Deadly (≥ +2).

Encounters are owner-only and can be loaded into a campaign's live view via `campaign-live-encounter.svelte`.

## Cross-references

- Reactive stores and the sync mechanism → [Frontend State](./frontend-state.md)
- Functions, permissions, and the campaign document → [Convex Backend](./convex-backend.md)
- Sharing homebrew into a campaign → [Homebrew](./homebrew.md)
- Routes for these pages → [Routing & Pages](./routing-and-pages.md)
