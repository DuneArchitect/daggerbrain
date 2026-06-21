# Plan: Get Subscriptions Live (Clerk Billing)

## Context

Daggerbrain gates paid features (`unlimited_characters`, `unlimited_homebrew`) behind
entitlements stored in Convex (`user_entitlements.feature_slugs`). The gating side is
fully built and working: `functions/entitlements.ts:getFeatures` reads the slugs and
`lib/state/user.svelte.ts` / `functions/characters.ts` enforce limits against them.

What's missing is the **source** of those entitlements. Billing is currently *disabled*:
`internal/entitlements.ts:refreshUserEntitlements` is a deliberate no-op, so no provider
ever writes entitlement rows. A Clerk billing webhook handler already exists in
`http.ts` but does nothing useful because of that no-op.

Decision (confirmed with user): use **Clerk Billing** (now GA; scaffolding already
Clerk-shaped) with a **single `adventurer` plan** granting both feature slugs. This is
the fastest path to live and reuses the existing webhook + entitlement abstraction.

> Note: `docs/billing.md` describes a *Stripe* target. This plan intentionally diverges
> toward Clerk Billing per the user's decision. `docs/billing.md` and `CLAUDE.md` should
> be updated to reflect Clerk Billing as the chosen provider (see step 6).

## Goal

A signed-in user can open an upgrade page, subscribe to `adventurer` via Clerk's hosted
checkout, and immediately have `unlimited_characters` + `unlimited_homebrew` unlocked —
because a Clerk webhook syncs their subscription into Convex `user_entitlements`.

## How the pieces connect

```
PricingTable (svelte-clerk)  ──checkout──▶  Clerk Billing
                                                 │ subscription.* webhook
                                                 ▼
                          http.ts /clerk/webhooks (verify + filter)  ──┐
                                                                       ▼
                  internal.entitlements.refreshUserEntitlements (internalAction)
                       └─ clerk.billing.getUserBillingSubscription(userId)
                       └─ collect active plan feature slugs
                       └─ runMutation upsertUserEntitlements
                                                 ▼
                                  Convex user_entitlements row
                                                 ▼
                 getFeatures → user.svelte.ts → gates unlock (no change needed)
```

## Configuration (Clerk Dashboard — manual, no code)

1. Enable **Billing** for the Clerk instance (uses Clerk's Stripe-backed gateway; dev
   uses test mode / test cards).
2. Create features with slugs **exactly** matching `src/convex/constants/entitlements.ts`:
   - `unlimited_characters`
   - `unlimited_homebrew`
3. Create plan with slug `adventurer` (matches `ADVENTURER_PLAN_SLUG`) and attach both
   features.
4. Create a **webhook endpoint** → `<CONVEX_SITE_URL>/clerk/webhooks`
   (local: `http://127.0.0.1:3211/clerk/webhooks`, prod: the Convex `.site` URL),
   subscribed to `subscription.*` and `subscriptionItem.*` events (the set already
   listed in `http.ts:RELEVANT_BILLING_EVENT_TYPES`). Copy the signing secret.

## Environment variables (Convex deployment, not just `.env.local`)

These run inside the Convex backend, so they must be set on the deployment via
`npx convex env set` (the `.env.local`/Convex split that already bit us with
`PUBLIC_CLERK_FRONTEND_API_URL`):

- `CLERK_WEBHOOK_SIGNING_SECRET` — read by `http.ts:36`.
- `CLERK_SECRET_KEY` — needed by the new `createClerkClient` call in the sync action.

## Code changes

### 1. `src/convex/internal/entitlements.ts` — re-enable the sync (core change)
Replace the no-op body of `refreshUserEntitlements` with:
- `import { createClerkClient } from '@clerk/backend'`
- `const clerk = createClerkClient({ secretKey: process.env.CLERK_SECRET_KEY })`
- `const sub = await clerk.billing.getUserBillingSubscription(clerkUserId)`
  (verified method on `BillingAPI`; returns `BillingSubscription`)
- Derive slugs: for each `sub.subscriptionItems` whose `status` is active, collect
  `item.plan?.features.map(f => f.slug)`; flatten + dedupe.
- `await ctx.runMutation(internal.internal.entitlements.upsertUserEntitlements, {
     clerkUserId, featureSlugs, syncedAt: Date.now() })`
- Handle "no active subscription" → `featureSlugs: []` (downgrade path).
- Wrap the Clerk call in try/catch; on error throw so Convex retries the action
  (do NOT silently swallow — a swallowed sync error means a paying user stays locked).

`upsertUserEntitlements` (same file) already does the insert/patch correctly — reuse as-is.

### 2. `src/convex/http.ts` — no logic change needed
Handler already verifies the signature, filters relevant events, extracts
`event.data.payer.user_id`, and calls `refreshUserEntitlements`. Confirm cancellation
events (`subscriptionItem.canceled`/`ended`) are in the filter set (they are) so
downgrades sync too.

### 3. New route `src/routes/(app)/subscribe/+page.svelte` — upgrade UI
- Render `<PricingTable />` from `svelte-clerk` (confirmed exported). Clerk hosts the
  full checkout. Wrap in the same auth/`Loader` pattern used in `profile/+page.svelte`.

### 4. Upgrade CTAs at the gates
- `src/routes/(app)/characters/+page.svelte`: when `character_limits.can_create_character`
  is false, show an "Upgrade to Adventurer" link → `/subscribe`.
- `src/routes/(app)/homebrew/+page.svelte`: same for `homebrew_limits.can_create_homebrew`.
- These already import the limits/constants; reuse `getUserContext()` state.

### 5. `src/routes/(app)/profile/+page.svelte` — manage subscription
- The page already renders `<UserProfile routing="hash" />`, which surfaces Clerk's
  billing/subscription management tab once Billing is enabled. Add a small status line
  using existing `features` from `getUserContext()` and a link to `/subscribe` for free
  users (uses `ADVENTURER_PLAN_SLUG` / `FREE_PLAN_SLUG` already imported there).

### 6. Docs
- Update `docs/billing.md` and the `CLAUDE.md` billing line to state Clerk Billing is the
  chosen provider (supersedes the Stripe-migration intent).
- Remove the now-unused `PUBLIC_STRIPE_PUBLISHABLE_KEY` from `.env.local` / `.env.example`
  (optional cleanup; it is read by zero code).

## What does NOT change
- Entitlement read path (`getFeatures`) and all UI gates (`user.svelte.ts`,
  `characters.ts`) — they already consume `feature_slugs` and are provider-agnostic.
- The `user_entitlements` schema.

## Verification (end-to-end)

1. `npx convex env set CLERK_WEBHOOK_SIGNING_SECRET ...` and confirm `CLERK_SECRET_KEY`
   is set on the deployment (`npx convex env list`).
2. Run `npm run convex:dev` + `npm run dev`.
3. **Unit-ish backend check (bypasses webhook tunnel):** after subscribing in Clerk test
   mode, run `npx convex run internal/entitlements:refreshUserEntitlements '{"clerkUserId":"<id>"}'`
   then `npx convex data user_entitlements` — confirm the row has the two slugs.
4. **Full flow:** as a free user, hit the character limit (create 6), see the upgrade CTA,
   go to `/subscribe`, complete Clerk test checkout (test card `4242 4242 4242 4242`).
   - Local webhook delivery needs a public URL: use an `ngrok`/tunnel to
     `127.0.0.1:3211` for the Clerk webhook endpoint, OR rely on step 3's manual refresh.
5. Confirm `getFeatures` now returns the slugs and the UI lets you create a 7th character
   / unlimited homebrew without reload (Convex reactivity pushes the update).
6. **Downgrade:** cancel the subscription in Clerk → webhook fires → row updates to `[]`
   → gates re-engage.

## Open risks / notes
- Clerk Billing API is marked `@experimental` (public beta). Pin `@clerk/backend` /
  `clerk-js` versions to avoid breaking changes (Clerk's own guidance).
- Local webhook testing requires a tunnel; the manual `convex run` path (step 3) keeps
  development unblocked without one.
