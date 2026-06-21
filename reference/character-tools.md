# Character Tools (Sheet, Editor, Derivation)

Characters are the heart of the app. There are three layers: the **derivation engine** (turns stored data into a playable sheet), the **sheet** (read/play view), and the **editor** (build/level-up view). All three read from the `character` store ([Frontend State](./frontend-state.md)).

Routes: `characters/+page.svelte` (list) → `characters/[id]/+page.svelte` (sheet) → `characters/[id]/(edit)/…` (editor group).

## The derivation engine

`src/lib/state/derive_character.ts` is a large **pure function**: `derive_character_state(character, compendium)` normalizes the stored character, then `derive_character_data` computes the full playable sheet. It is pure so the same inputs always produce the same sheet, and so it can run both in the UI store and in the [PDF export](#pdf-export).

### Why it loops

Some derived values depend on each other: subclass upgrade choices set **mastery levels**, mastery levels unlock **features**, features add **modifiers**, modifiers change **traits / proficiency / loadout size**, which can change which cards are in the loadout, which changes active modifiers again. The engine resolves this with an **iterative stabilization loop** (up to 3 passes) that re-derives loadout → modifiers → proficiency/traits/mastery until the result stops changing.

### What it computes

- **Scalars** (each via a `base → bonus → override` modifier triplet): `max_hp`, `max_stress`, `max_hope` (minus scars), `max_burden`, `max_loadout`, `max_experiences`, `evasion`, `max_armor` (capped 12), `proficiency`, `spellcast_roll_bonus`. Several get level bumps at levels 2/5/8.
- **Damage thresholds** from equipped armor (or level-based defaults), with level-bump logic when modified.
- **Traits** (agility, strength, finesse, instinct, presence, knowledge): base + level-up marks + class/beastform bonuses + modifiers.
- **Derived items**: equipped primary/secondary weapon, armor, unarmed attack — each with weapon-modifier passes and special handling (e.g. Combat Training sets burden to 0 and adds level to physical damage).
- **Beastform** and **Companion** (when the relevant class/subclass feature is present), with their level-up customizations.
- **Domain card vault** (everything available) and **loadout** (what's active, bounded by `max_loadout`, honoring `forced_in_loadout`).
- **Feature flags** like `hasBeastformClassFeature`, `hasCompanionSubclassFeature`, `hasRallyClassFeature`, `hasPrayerDiceClassFeature`, etc., used to conditionally render sheet sections.

### Normalization (self-healing)

Before deriving, the character is cleaned across several passes so that data invalidated by edits/deletions can't break the sheet: clear level-up choices above current level, validate subclass/class relationships and multiclass rules, drop inventory/equipment whose compendium entry is missing or whose level requirement isn't met, sanitize card choices and beastform/evolution selections, seed class background questions, and clamp marked HP/stress/hope/armor and loadout to current maxima.

## The sheet

`src/lib/components/character-sheet/character-sheet.svelte` orchestrates a responsive grid of **standalone widgets**, a tabbed **features** panel, and a **slideover** for detail views.

- **Standalone widgets** (`standalone/`): `hp`, `stress`, `hope` (+ scars), `evasion`, `armor-slots`, `damage-thresholds`, `traits`, `experiences`, `gold`, `character-portrait`, `revive-button`. Each renders one derived resource and writes marked values back to the store.
- **Features tabs** (`features/character-features.svelte`, tab choice persisted to localStorage): Weapons & Armor, Class Features (per-class components — bard/guardian/seraph/wizard), Beastform*, Companion*, Inventory, Background, Notes (* shown only when the feature flag is set).
- **Cards** (`cards/`): `character-cards` (ancestry/community/transformation) and `loadout` (active domain cards, reorderable).
- **Companion** (`companion/`): display + edit modes with its own evasion/stress/hope and level-up choices.
- **Campaign section** (`campaign/`): fear track, countdowns, and dice log surfaced from the active campaign.

### Slideover detail system

`slideover/slideover-sheet.svelte` is a right-hand panel that swaps its body based on a `SlideoverContent` union. Opening an item (weapon, armor, consumable, loot, domain/heritage card, beastform, experience, conditions, scars, downtime, death move, sheet customization, equipment catalog…) renders the matching component from `slideover/content/`. Many detail panels embed a contextual rule reminder from `rule-snippets/` (weapons, armor, loot, consumable, conditions, downtime, scars, death move, experience).

## The editor

The `(edit)` route group shares one layout (`(edit)/+layout.svelte`: tab bar, portrait/name editing, prev/next navigation). Each tab is a page:

| Page | What it edits |
|---|---|
| `edit` | name / description (home tab) |
| `heritage` | ancestry, community, (transformation) card pickers + their choices |
| `class` | the **leveling system** — level select + tier 1–4 option groups |
| `traits` | allocate the six trait modifiers (no duplicates), suggested-traits helper |
| `experiences` | add/edit experiences and their level-up bonuses |
| `equipment` | starting equipment, active armor/weapons, inventory, equipment catalog |

### Leveling system (`character-editor/leveling/`)

`tier-options-group.svelte` is the generic renderer; `tier-1-options` … `tier-4-options` configure it per tier (tier 1: class + subclass + level-1 cards; tiers 3–4 add the exclusive multiclass-vs-subclass-upgrade choice). The **secondary-options/** selectors are the actual pickers: primary/secondary class, primary/secondary subclass, domain-card selector, traits selector, and a class summary. `domain-card-utils.ts` filters available/previously-chosen cards; `highlight-utils.ts` drives beastform-bonus highlighting.

### Supporting component groups

- `character-editor/equipment/` — active armor/weapon slots, inventory list, starting equipment.
- `character-editor/background/` — background questions, descriptions, connections.
- `catalogs/` — searchable/filterable pickers used throughout (equipment, domain-card, heritage-card, card, adversary, beastforms, environment). Each takes an `onSelect` callback.
- `compendium-items/` — the read-only renderers for each item type (cards, equipment details, adversary, beastform, environment), reused by catalogs, slideovers, and homebrew previews.
- `conditions/condition-chip.svelte` — a condition badge shown in the sheet header.

## PDF export

`src/lib/pdf/character-sheet-export.ts` uses **pdf-lib** to fill a fillable AcroForm template (`/pdfs/blank-character-sheet.pdf`). It takes the character + derived data, maps stats/traits/equipment/domain-cards/companion to form fields, sanitizes text to PDF-safe ASCII, strips HTML from descriptions, and embeds images (portrait, class icon) as PNG (rasterizing SVG icons to white). This is the second consumer of the derivation output, reinforcing why derivation is a pure function.

## Cross-references

- The store that owns `character` and runs derivation → [Frontend State](./frontend-state.md)
- The content the sheet/editor resolve against → [Compendium & SRD](./compendium-and-srd.md)
- Campaign fear/countdown/dice surfaced on the sheet → [Campaigns & Live Play](./campaigns-and-live-play.md)
- UI primitives and theming → [UI & Theming](./ui-and-theming.md)
