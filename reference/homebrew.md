# Homebrew Authoring

Homebrew lets users create their own versions of all [15 compendium item types](./compendium-and-srd.md#the-15-item-types). Its defining quality is **uniformity**: every type follows the exact same four-part pattern, so understanding one type means understanding all fifteen.

## The per-type pattern

For each type there is a folder `src/lib/components/homebrew/forms/<type>/` containing three files, plus a matching preview:

| File | Responsibility |
|---|---|
| `form.svelte` | the edit UI — `sveltekit-superforms` bound to the type's Zod schema via `zod4Client(Schema)` |
| `normalize.ts` | shape conversion: `<Type>ToFormData`, `<Type>FormDataToItem`, `normalize<Type>` |
| `errors.ts` | `summarize<Type>FormErrors` — maps field paths to human labels |
| `../previews/<type>-preview.svelte` | live read-only render shown beside the form |

`normalize.ts` is where save-time hygiene happens: trim/sanitize strings (HTML stripped via DOMPurify), coerce floats to ints for level/burden/bonus fields, dedupe arrays, **stamp `source_key: 'Homebrew'`**, recurse into nested features/modifiers/conditions, and finally `Schema.parse()` the result. The Zod schema is shared with the Convex backend (see [the zodToConvex pattern](./convex-backend.md#schema-and-the-zodtoconvex-pattern)), so client and server validate against the same definition.

The 15 types: `adversary, ancestry-card, armor, beastform, class, community-card, consumable, domain, domain-card, environment, loot, primary-weapon, secondary-weapon, subclass, transformation-card`.

## Shared form fragments (`forms/shared/`)

Composable pieces reused across types — this is how 15 forms stay DRY:

- **`features/`** — the feature-list editor (title + HTML description + nested modifiers); `defaults.ts` provides `emptyFeature`, `emptyCharacterModifier`, `emptyWeaponModifier`.
- **`character-modifier/`** & **`weapon-modifier/`** — a single modifier row (target stat, behavior `base`/`bonus`/`override`, value type), each nesting conditions.
- **`character-conditions/`** & **`weapon-conditions/`** — when a modifier applies (level range, armor/weapon equipped, melee/ranged, card-choice gates).
- **`card-choices/`** — arbitrary or experience-based card option selections.
- **`helpers.ts`** — error-path machinery: `summarizeSuperformErrors`, `parseSuperformPath` (`"features[0].title"` → `['features', 0, 'title']`), and path filters (`homebrewErrorsAt`, `firstHomebrewErrorAt`, `homebrewHasErrorsBelow`). These let each type's `errors.ts` report nested errors with friendly labels like "Feature 1: Trait".

## Routing & dynamic dispatch

- `homebrew/+page.svelte` — listing + creation. Lists all types in collapsible tabs with per-type counts, enforces the homebrew cap (20 free, unlimited with the entitlement), and offers filters (search/type/tier/weapon/domain).
- **Creation flow:** pick a type → `template-combobox.svelte` offers an SRD/official item as a starting point (or "start from scratch" using `COMPENDIUM_DEFAULTS[type]`) → deep-clone, set title → `homebrew.addItem` → redirect to the editor.
- `homebrew/[type]/[uid]/+page.svelte` — the editor. The `[type]` param **switches** to the right form + preview component (form on the left, live preview on the right). Save calls `homebrew.updateItem`; a `beforeNavigate` guard warns on unsaved changes; back/next navigates between items of the same type. The `[uid]/+layout.svelte` validates the type and that the item exists in the loaded vault.

`template-combobox` groups templates intelligently: by **tier** (weapons/armor/beastforms/adversaries/environments), by **class** (subclasses), by **domain + level** (domain cards), or flat (everything else), badging homebrew entries with the anvil `homebrew-badge`.

## Backend (`src/convex/functions/homebrew.ts`)

Items live in 15 per-type tables (`primary_weapons`, `armor`, …), each row `{ owner_clerk_id, item }`. Operations:

- `listIds` / `get` — read (with `getHomebrewAccess`: owner edits, shared-campaign members read).
- `add` — enforces the cap, inserts with `owner_clerk_id`, appends the id to the user's `homebrew_vault[type]`, increments `homebrew_count`.
- `update` — requires `canEdit`; patches the normalized item.
- `remove` — requires `isOwner`; deletes and removes the id from the vault, decrements the count.

See [Convex Backend → Entitlements](./convex-backend.md#entitlements-paid-access) for the cap logic.

## State (`src/lib/state/homebrew.svelte.ts`)

`createHomebrew()` subscribes (via `convexClient.onUpdate`) to every id in the user's `homebrew_vault` and assembles a live `CompendiumContent`, exposing `isLoading`/`error`/`compendium` plus `addItem`/`updateItem`/`removeItem`. It is the same vault-subscription machinery used for campaigns (the compendium-vault factory in [Frontend State](./frontend-state.md)).

## Campaign vault integration

A GM shares homebrew into a campaign so the whole table can use it:

1. User creates items → stored in their `homebrew_vault`.
2. In the campaign, `campaign-vault.svelte` + `homebrew-items-combobox.svelte` pick items to add.
3. The selected ids are written into `campaign.homebrew_vault`.
4. Characters in that campaign resolve those items under `source_key: 'Campaign'` (see [how content layers combine](./compendium-and-srd.md#sources-and-how-content-layers-combine)).

Ownership never transfers — the campaign vault references items, it doesn't copy or claim them.

## Cross-references

- Item schemas & shared building blocks → [Compendium & SRD](./compendium-and-srd.md)
- Form UI primitives → [UI & Theming](./ui-and-theming.md)
- Backend tables, ownership, caps → [Convex Backend](./convex-backend.md)
