# Convex Backend

The backend lives entirely in `src/convex/` and runs on [Convex](https://convex.dev). It is the **source of truth for application access** — feature gates and permissions are enforced here, never trusted from the client.

> When editing anything in this directory, read `src/convex/_generated/ai/guidelines.md` first. It contains Convex API rules that override general knowledge.

## Directory layout

```
src/convex/
├── schema.ts                 # defineSchema: all tables + indexes
├── schemas/                  # Zod schemas (domain types) -> reused by validators + forms
│   ├── characters.ts         # CharacterSchema, InventorySchema, CompanionSchema
│   ├── campaigns.ts          # CampaignSchema, CampaignMemberSchema, CampaignCharacterSchema
│   ├── compendium.ts         # All 15 compendium item schemas + CompendiumContent(Ids)Schema
│   ├── encounters.ts         # EncounterSchema, AdversaryInstanceSchema
│   ├── users.ts              # UserSchema (campaign_ids, counts, homebrew_vault)
│   ├── rules.ts              # TraitId, DamageType, Range, Tier, Feature, Countdown, SourceKey...
│   ├── dice.ts               # DiceHistorySchema, RollSchema
│   ├── sources.ts            # SourceMetadataSchema (legacy sources table)
│   └── themes.ts             # re-exports lib/schemas/themes
├── functions/                # PUBLIC api surface (query / mutation)
│   ├── campaigns.ts          # campaign CRUD, membership, roster, dice history
│   ├── characters.ts         # character CRUD + compendium-scope-for-view
│   ├── encounters.ts         # encounter CRUD (owner-only)
│   ├── homebrew.ts           # homebrew CRUD across all 15 item tables
│   ├── entitlements.ts       # getFeatures (read user's feature slugs)
│   ├── sources.ts            # list unlocked official source metadata
│   ├── streamOverlays.ts     # stream overlay token + state
│   └── users.ts              # get / create user record
├── internal/                 # NOT exposed to client
│   ├── entitlements.ts       # refreshUserEntitlements (action), upsertUserEntitlements
│   └── helpers.ts            # clearAllData (dev only)
├── lib/
│   └── characterCompendium.ts # resolve which content a character can "see"
├── constants/
│   ├── entitlements.ts       # feature slugs, free-tier limits, default sources
│   ├── constants.ts          # default documents for new characters/encounters/campaigns
│   └── rules.ts              # game-rule constants
├── permissions.ts            # access-resolution helpers (ownership / membership)
├── http.ts                   # httpRouter: Clerk billing webhook
├── auth.config.ts            # Clerk JWT provider config
└── _generated/               # codegen (api, dataModel, server) — do not edit
```

## Schema and the `zodToConvex` pattern

Domain types are written **once in Zod** (`src/convex/schemas/*`) and converted to Convex validators with `zodToConvex` from `convex-helpers/server/zod4`. This single source of truth feeds:

- Convex table definitions and function arg validators (server)
- SvelteKit form validation via `sveltekit-superforms` + Zod (client)

```ts
// schema.ts
characters: defineTable({
  owner_clerk_id: v.string(),
  campaign_id: v.optional(v.id('campaigns')),
  character: zodToConvex(CharacterSchema)        // whole entity stored as one nested object
}).index('by_owner_clerk_id', ['owner_clerk_id'])
```

A recurring modeling choice: **rich entities are stored as a single nested object** (`character`, `campaign`, `item`, `encounter`) rather than normalized columns. This keeps the dense TTRPG data together and lets the same Zod type validate it end to end.

### Tables

| Table | Stores | Index(es) | Notes |
|---|---|---|---|
| `characters` | `owner_clerk_id`, optional `campaign_id`, `character` object | `by_owner_clerk_id` | one owner; campaign members can read |
| `encounters` | `owner_clerk_id`, `encounter` object | `by_owner_clerk_id` | owner-only |
| `campaigns` | `invite_code`, `campaign`, **`members[]`**, **`characters[]`** | `by_invite_code` | membership + roster are **embedded arrays**, not join tables |
| `dice_history` | `campaign_id`, `history` | `by_campaign_id` | one row per campaign, capped at ~50 rolls |
| `stream_overlays` | `campaign_id`, `token`, `enabled`, `modules`, `settings`, `layout` | `by_campaign_id`, `by_token` | public overlay config |
| `users` | `clerk_id` + `UserSchema` (campaign_ids, counts, homebrew_vault) | `by_clerk_id` | denormalized `character_count` / `homebrew_count` |
| `user_unlocked_sources` | `clerk_id`, `unlocked_source_keys[]` | `by_clerk_id` | which official sources a user owns |
| `user_entitlements` | `clerk_user_id`, `feature_slugs[]`, `synced_at` | `by_clerk_user_id` | paid capability flags |
| `sources` | legacy official-source content | `by_source_key` | **deprecated** — see compatibility note below |
| 15 homebrew tables | `owner_clerk_id`, `item` object | `by_owner_clerk_id` each | `primary_weapons`, `secondary_weapons`, `armor`, `loot`, `consumables`, `beastforms`, `classes`, `subclasses`, `domains`, `domain_cards`, `ancestry_cards`, `community_cards`, `transformation_cards`, `adversaries`, `environments` |

> **Compatibility note (in `schema.ts`):** the `sources` table is retained only for legacy deployments. Official source data has moved from Convex into local app build modules (`src/compendium/SRD`). See [Compendium & SRD](./compendium-and-srd.md).

## Authentication

Configured in `auth.config.ts` as a Clerk JWT provider. Every authenticated function resolves identity through:

```ts
const identity = await ctx.auth.getUserIdentity();
// identity.subject  -> the stable Clerk user id, used as owner_clerk_id / clerk_id
```

There is no separate user table key today: **`identity.subject` (the Clerk subject id) is the canonical user key**, stored as `owner_clerk_id` on owned content and `clerk_id` on the user record. (The longer-term intent — migrating to internal Convex `users._id` — is documented in [`docs/permissions.md`](../docs/permissions.md) and [`docs/billing.md`](../docs/billing.md).)

## Permissions (`permissions.ts`)

Access is enforced server-side. Helpers return a typed access object or `null` (no access). Functions call a helper first, then mutate only if allowed.

| Helper | Resolves | Returns |
|---|---|---|
| `getCampaignAccess(ctx, id)` | membership in `campaign.members` | `{ campaign, members, characters, invite_code, isOwner }` — `isOwner` when role is `GM` |
| `getCharacterAccessDetails(ctx, id)` | owner, else campaign-member of the character's campaign | `{ canEdit, isOwner, owner_clerk_id, ... }` — owners and GMs `canEdit` |
| `getCharacterAccess(ctx, id)` | wrapper around the above (drops `owner_clerk_id`) | character access |
| `getEncounterAccess(ctx, id)` | owner only | `{ encounter, isOwner }` |
| `getHomebrewAccess(ctx, id)` | owner (edit) or shared-campaign member (read) | `{ item, canEdit, isOwner }` |

The model in one sentence: **owners mutate; campaign membership grants read (and GMs read+edit characters); nothing else gets in.** Homebrew read access is intentionally broader than edit access — sharing a campaign can make homebrew visible, but only ownership grants mutation.

## Public functions (`functions/`)

| File | Queries | Mutations |
|---|---|---|
| `campaigns.ts` | `get`, `resolveInvite`, `list`, `getDiceHistory` | `add`, `update`, `changeDisplayName`, `remove`, `leave`, `join`, `rotateInviteCode`, `addCharacter`, `removeCharacter`, `claimCharacter`, `unassignCharacter`, `updateDiceHistory` |
| `characters.ts` | `list`, `get`, `getCompendiumScopeForView` | `add`, `update`, `remove` |
| `encounters.ts` | `list`, `get` | `add`, `update`, `remove` |
| `homebrew.ts` | `listIds`, `get` | `add`, `update`, `remove` |
| `entitlements.ts` | `getFeatures` | — |
| `sources.ts` | `list` | — |
| `streamOverlays.ts` | `getForCampaign`, `getOverlayState` | `createOrRotate`, `updateSettings` |
| `users.ts` | `get` | `create` |

Highlights:

- **`characters.add` / `campaigns.claimCharacter`** enforce the character cap (see Entitlements).
- **`homebrew.add`** stamps `source_key = 'Homebrew'`, enforces the homebrew cap, and updates the user's `homebrew_vault` + `homebrew_count`.
- **`campaigns.add`** creates the campaign with a random 12-char `invite_code`, seeds a `GM` membership for the caller, and initializes an empty `dice_history`.
- **`campaigns.addCharacter`** appends to the embedded `characters[]` array; a GM adds `status: 'unclaimed'`, a player adds `status: 'active'` claimed by themselves.
- **`campaigns.claimCharacter`** transfers character ownership to the claiming player (one claimed character per player per campaign).
- **`streamOverlays.getOverlayState`** is looked up by `token` (no Clerk auth) and returns only the modules/state the overlay enables.

## Entitlements (paid access)

Defined in `constants/entitlements.ts`; gates live in the functions above.

| Slug | Free-tier limit | Gated by |
|---|---|---|
| `unlimited_characters` | 6 characters | `characters.add`, `campaigns.claimCharacter` |
| `unlimited_homebrew` | 20 homebrew items | `homebrew.add` |

- Default unlocked sources for every user: `['SRD']`.
- A user's slugs live in `user_entitlements.feature_slugs`; `entitlements.getFeatures` exposes them to the client for **UI hinting only** (the server re-checks on every gated mutation).
- The longer-term billing intent (Clerk auth + Stripe billing, Convex as the entitlement source of truth) is documented in [`docs/billing.md`](../docs/billing.md).

## HTTP endpoints (`http.ts`)

| Method | Path | Handler | Purpose |
|---|---|---|---|
| `POST` | `/clerk/webhooks` | `clerkWebhook` (httpAction) | verifies the Clerk webhook signature, extracts the Clerk user id, and calls `internal.internal.entitlements.refreshUserEntitlements` to re-sync feature slugs |

Returns `200` on success/ignored events, `400` on bad signature, `500` on misconfiguration. The refresh path is currently a no-op pending the Stripe billing migration.

## Internal functions (`internal/`)

- `entitlements.refreshUserEntitlements` (internalAction) + `upsertUserEntitlements` (internalMutation) — billing-sync plumbing.
- `helpers.clearAllData` (internalMutation) — destructive dev-only reset.

## Compendium scope resolution (`lib/characterCompendium.ts`)

When a character is viewed, the app must resolve *which content the character is allowed to use*. This module computes the **scope** as three layers:

1. `source_keys` — official sources the **character owner** has unlocked,
2. `homebrew_vault` — the owner's homebrew item ids,
3. `campaign_vault` — homebrew shared into the character's campaign.

This is **owner-scoped, not viewer-scoped**: a GM viewing a player's sheet resolves the *player's* available sources so the sheet renders correctly. See `getCharacterCompendiumScopeForView`.

## Cross-references

- Data consumed reactively on the client → [Frontend State](./frontend-state.md)
- How content layers (SRD + homebrew + campaign) combine → [Compendium & SRD](./compendium-and-srd.md)
- Campaign collaboration semantics → [Campaigns & Live Play](./campaigns-and-live-play.md)
- Auth wiring into SvelteKit → [Routing & Pages](./routing-and-pages.md)
