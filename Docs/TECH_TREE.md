# Tech Tree

The plugin gates production and research through **one** decision point (`URTSTechLibrary`), so the
command card, tooltips, AI, and server authority all agree. **You author the tree as DATA — zero C++.**

## What you can express
1. **Building/unit prerequisites** — "requires a Barracks" (existing `URTSRequirementsComponent.RequiredActors`).
2. **Upgrade → upgrade prerequisites, with levels** — "Air Weapons L2 requires Air Armor L1".
   (The self-chain — L2 needs L1 of the *same* key — is automatic; you never author it.)
3. **Per-level structure gates** — "an Armory is required for Weapons L2 & L3 but not L1".
4. **Tier gating** — "requires you own a Tier-2 building (a Lair)", which **re-locks live** if that
   building dies — exactly like Brood War.

## Authoring (all in the Blueprint/data editor)

**Upgrade prerequisites** — on a research building's `URTSResearchComponent → Available Research → [option]`:
- **Upgrade Prerequisites** — array of `{ Upgrade Key, Min Level }` cross-key gates.
- **Required Actors** — building/unit classes gating the *whole* line (any level).
- **Required Building For Level** — a map `{ level → building class }` (e.g. `2 → BP_Armory`, `3 → BP_Armory`).

**Tier gating** — drop a **`RTS Tier`** component (`URTSTierComponent`) on a Blueprint:
- On a *building* → set **Tier Level** (a Lair is 2, a Hive is 3). Owning a ready one "reaches" that tier.
- On the *thing being gated* (unit or research building) → set **Required Tier** (0 = no gate).

Nothing is replicated anew — gating reads owned actors + the already-replicated upgrade levels, so every
client shows the same lock state, and the server re-checks on `StartResearch` (authoritative).

## What you get automatically (no wiring)
- **Command card**: a gated research/production button greys out (dim frame, 0.3 icon, a **REQ lock
  badge**) with a **"REQUIRES ARMORY" / "REQUIRES TIER 2"** tooltip. It **unlocks the instant** you build
  the gate (0.25s poll), and re-locks if a tier building is lost.
- **AI**: never attempts a locked upgrade (it funnels through the same `CanResearch`), and climbs chains
  in order (L1 before L2).
- **Server authority**: a locked research is rejected even from a hacked client.
- **Cancellation**: active research exposes a bundled cancel command. The server returns the exact
  paid cost multiplied by the option's authored `Cancel Refund Factor` (75% by default).

## Query it from your own Blueprints
`URTSTechLibrary` (BlueprintPure, static):
- `Get Missing Requirement For Upgrade (WorldCtx, OwnedActor, UpgradeKey, TargetLevel)` → `FRTSTechRequirementResult`
  (`bBlocked`, `GateKind`, `Reason` text, blocking class/key/level).
- `Get Missing Requirement For Product (WorldCtx, OwnedActor, Product)` → same result.
- `Can Perform Upgrade` / `Can Perform Product` → bool shortcuts.
- `URTSResearchComponent::Get Research Block Reason (UpgradeKey)` → the result for a research building's next level.

Bind a custom tech-tree screen to `URTSUpgradeComponent::OnUpgradeLevelChanged` +
`URTSResearchComponent::OnResearchCompleted` to re-evaluate node states on change (both already replicate).
Use `IssueCancelResearchOrder` from an owning player widget; trusted server logic can call the
authority-only `CancelResearch` event directly. `CanResearch`, next-level cost/time, start, and
cancel all have Blueprint Native Event defaults for Blueprint and C++ policy overrides.

## Author a test gate (Brood War Infantry Weapons)
On the research building's `Weapon.*` option: `Max Level = 3`, `Required Building For Level = { 2: BP_Armory, 3: BP_Armory }`.
Result: research L1 freely; after L1, L2 shows **REQUIRES ARMORY** until an Armory exists, then unlocks; L3
likewise. Add `URTSTierComponent{RequiredTier=2}` to a unit + `URTSTierComponent{TierLevel=2}` to a Lair to
see tier gating (and live re-lock when the Lair dies).

## Design notes
Prerequisite data lives on the **existing** `FRTSResearchOption` / `URTSRequirementsComponent` — there is
**no separate tech-graph asset to keep in sync**, no new subsystem, and no new replicated state. Costs/times
still come from the same `FRTSResearchOption` fields. Skinning flows from `URTSHudStyle` (the lock frame,
badge, and REQ tooltip are the same tokens as building requirements).
