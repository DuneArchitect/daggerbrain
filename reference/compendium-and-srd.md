# Compendium & SRD Game Data

The "compendium" is all the Daggerheart game content — weapons, armor, classes, domain cards, adversaries, environments, etc. There are **15 item types**, and they are the spine of the whole app: the same 15 appear as Convex schemas, database tables, homebrew forms, and preview components.

## The 15 item types

| Type | Notable fields |
|---|---|
| `primary_weapons` / `secondary_weapons` | level_requirement, type (Physical/Magical), range, damage_dice, burden, traits, features |
| `armor` | max_armor, damage_thresholds {major, severe}, features |
| `loot` | rarity_roll (1–60), character/weapon modifiers |
| `consumables` | rarity_roll (1–60), effect description |
| `beastforms` | category, level_requirement, character_trait bonus, attack, advantages, evasion_bonus, special_case |
| `classes` | starting HP/evasion, hope_feature, **primary/secondary domain ids**, class_features, **subclass_ids**, suggested gear, starting inventory, background/connection questions |
| `subclasses` | class_id, spellcast_trait, **foundation/specialization/mastery cards** (each with level-up options) |
| `domains` | title, color, foreground_color, art |
| `domain_cards` | **domain_id**, level_requirement, recall_cost, category (ability/spell/grimoire), features, options |
| `ancestry_cards` | features, options, optional is_mixed_ancestry |
| `community_cards` | features, options |
| `transformation_cards` | features, options (Void content; SRD set is empty) |
| `adversaries` | tier (1–4), type (Bruiser/Horde/Leader/Minion/Ranged/Skulk/Social/Solo/Standard/Support), thresholds, attack, features, experiences |
| `environments` | tier, type (Exploration/Social/Traversal/Event), impulses, potential adversaries, features |

The Zod schemas for all of these are in **`src/convex/schemas/compendium.ts`**; shared building blocks (`BaseCard`, `Feature`, `CharacterModifier`, `WeaponModifier`, `TraitId`, `Tier`, `Range`, `SourceKey`) are in **`src/convex/schemas/rules.ts`**. A `CompendiumContent` is just a record-of-records:

```ts
type CompendiumContent = {
  primary_weapons: Record<string, PrimaryWeapon>;
  // … one keyed record per type …
  environments: Record<string, Environment>;
};
```

Items are keyed by lowercase snake_case ids (`bard`, `gambeson_armor`, `rune_ward`, `drakona`).

## SRD data is bundled, not stored

The official **System Reference Document** content lives as TypeScript modules under `src/compendium/SRD/` and is **compiled into the bundle** — it is *not* fetched from Convex at runtime.

```
src/compendium/SRD/
├── index.ts                     # exports SRD_SOURCE_METADATA, SRD_COMPENDIUM
└── compendium/
    ├── index.ts                 # aggregates all 15 categories -> CompendiumContent
    ├── primary-weapons/  tier-1..4 files -> index
    ├── armor/            tier-1..4 files -> index
    ├── beastforms/       tier-1..4 files -> index
    ├── adversaries/      tier-1..4 files -> index
    ├── domain_cards/     arcana|blade|bone|codex|grace|midnight|sage|splendor|valor -> index
    ├── classes/  subclasses/  domains/   (single index each)
    ├── loot/  consumables/  environments/  (single index each)
    └── ancestry-cards/  community-cards/  transformation-cards/
```

Each leaf file exports a record of items; each `index.ts` spreads its children upward; the top `compendium/index.ts` assembles the full `CompendiumContent`.

> **History:** a legacy `sources` table still exists in `schema.ts` (with a removal note). Official data was moved out of Convex into these build modules on 2026-05-06 (commit `010ab4a`). New deployments rely on the bundled SRD.

## Sources, and how content layers combine

A `SourceKey` is one of: `'SRD' | 'The Void 1.5' | 'Campaign' | 'Homebrew'`.

- **Official sources** (`SRD`, and the planned `The Void 1.5`) are bundled and gated by what a user has unlocked. Every user defaults to `['SRD']` (`DEFAULT_UNLOCKED_SOURCES` in `src/convex/constants/entitlements.ts`); unlocks are stored in the `user_unlocked_sources` table. Metadata + merge live in `src/lib/compendium/official-sources.ts` (`getOfficialCompendiumFromSourceKeys`).
- **Homebrew** is the user's own items (stamped `source_key: 'Homebrew'`), enabled per-character.
- **Campaign** is homebrew the GM shared into a campaign's `homebrew_vault` (stamped `source_key: 'Campaign'`).

A character's usable content is the **merge** of these layers, assembled in `character.svelte.ts`:

```ts
const compendiums = [getOfficialCompendiumFromSourceKeys(enabledOfficialSourceKeys)];
if (available_source_keys.includes('Homebrew')) compendiums.push(ownerHomebrewVault.compendium);
if (available_source_keys.includes('Campaign')) compendiums.push(campaignVault.compendium);
const full_character_compendium = merge_compendium_content(...compendiums);
```

`merge_compendium_content` (in `src/lib/utils.ts`) shallow-merges each keyed record; later layers win on key collision. The server-side scope (which sources/vaults a viewer may resolve, owner-scoped) is computed in `src/convex/lib/characterCompendium.ts`.

## Domains, classes, subclasses (relationships)

- **9 domains:** arcana, blade, bone, codex, grace, midnight, sage, splendor, valor — each with a color used for banners/icons.
- **Classes** reference a `primary_domain_id` and `secondary_domain_id` (must differ); choosing a class grants access to both domains' cards. Classes list their `subclass_ids`.
- **Domain cards** carry a `domain_id`, a `level_requirement`, a `recall_cost`, and a `category`.
- **Subclasses** belong to a `class_id`, optionally define a `spellcast_trait`, and contain three progression cards (foundation → specialization → mastery), each able to grant level-up domain-card options.

These relationships are what the [derivation engine](./character-tools.md#the-derivation-engine) walks to assemble a character's available cards, traits, and features.

## Cross-references

- How these schemas become DB tables and validators → [Convex Backend](./convex-backend.md#schema-and-the-zodtoconvex-pattern)
- How users author their own items → [Homebrew](./homebrew.md)
- How the merged compendium drives a character → [Character Tools](./character-tools.md)
