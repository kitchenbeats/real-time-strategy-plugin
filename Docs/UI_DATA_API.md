# RealTimeStrategy Plugin — UI DATA API (Binding Cookbook)

**The contract licensees bind their own HUDs to.** Every getter below is `BlueprintPure`, every
state-change event is a `BlueprintAssignable` delegate, and everything new added for this contract
lives under the Blueprint category **`RTS|UI Data`**. Widgets never poll per-frame except where this
document explicitly says "poll" (local-only data with no delegate — supply, control groups).

The presentation contract is `URTSHudStyle`. `URTSEventFeedDemoWidget` under `Public/Feedback/` is
the native reference for building a Blueprint or C++ presentation layer without a project-specific
asset-generation script.

---

## 0. Presentation policies and supported limits

Everything in the mockup matrix (§1) is backed by a real getter/delegate. The remaining limits are
intentional product policies rather than missing data:

| Policy | Supported behavior |
|---|---|---|
| **Team swatch color** | `URTSHudStyle::TeamColors` resolves the replicated team index. Projects can replace the palette without changing gameplay. |
| **Rank chevron tiers** | `URTSHudStyle::RankKillThresholds` maps replicated kills to presentation tiers (default 1/3/6). |
| **Allied player pings** | `ARTSPlayerController::IssueAlliedPing` uses the replicated `RTSPingComponent` for validated, rate-limited, authority-filtered delivery. Sender and recipient policies are Blueprint/C++ overridable. |
| **Sim-speed of other human players** | Remote controllers intentionally do not exist on clients. The match-strip marker covers the local player and server-side AI/observer data without replicating private controller state. |

---

## 1. Mockup element → data source matrix

Legend — **EXISTING**: was already exposed to Blueprint. **ADDED**: additively exposed/added by this
pass (specifier, thin getter, or assignable delegate on an existing system). **NEW**: new tracking
implemented by this pass (data that did not exist anywhere before).

| # | Mockup element (IRONFRAME) | Data source (component → member) | State | Event / refresh | Replication |
|---|---|---|---|---|---|
| 1 | Resource counter values (`1,240` / `655`) | `URTSPlayerResourcesComponent` (on the **controller**) → `GetResources(Type)`; chip set from `GetResourceTypes()` | EXISTING; `GetResourceTypes` **ADDED** (was C++-only) | `OnResourceCatalogChanged` rebuilds chips; `OnResourcesChanged(Type, Old, New, bSyncedFromServer)` updates exact balances | Catalog and balances replicate owner-only; `bSyncedFromServer` distinguishes server echo (suppress spend-flash on it) |
| 2 | Income sub-chips (`+824/min`) | `URTSEventFeedSubsystem::GetIncomePerMinute(Type)` — 60 s rolling window of positive stock deltas | **NEW** | Recompute on each `OnResourcesChanged`, or on a slow (1 Hz) tick | Local derivation of replicated stock |
| 3 | Supply chip (`53/58`, NEAR CAP) | `URTSGameplayLibrary::GetPlayerSupply(WCO, Controller, Used, Cap)`; per-actor `URTSSupplyComponent::GetSupplyCost/GetSupplyProvided` | EXISTING | **Poll 4 Hz** (sums replicated actors; no delegate) | Client-safe |
| 4 | Supply-blocked flash / toast | `ARTSPlayerController::OnSupplyBlockedEvent(ProductClass)`; feed `HUDEVENT_SupplyBlocked` (10 s throttle) | **ADDED** (delegate + client-side supply check in `CheckCanIssueProductionOrder`) | Event | Local (fires where the order was issued) |
| 5 | Upgrade pips (`W2 A1 S0`) | `URTSUpgradeComponent` (on the controller) → `GetUpgradeLevel(Key)`; per-actor static `GetUpgradeLevelForActor(Actor, Key)` | EXISTING | `OnUpgradeLevelChanged(Key, Old, New)` (EXISTING) | `Upgrades` replicates; delegate fires on clients via RepNotify |
| 6 | Match clock (`12:47`) | `ARTSGameState::GetMatchTimeSeconds()` | **NEW** (replicated `MatchStartServerTime`) | Poll 1 Hz (it's a clock) | Server time — all clients agree |
| 7 | Player name / team swatch | `APlayerState::GetPlayerName()`; `ARTSPlayerState::GetTeam()` → `ARTSTeamInfo::GetTeamIndex()` | EXISTING | `ReceiveOnTeamChanged` | Replicates |
| 8 | Sim-speed marker (`×1.5`) | `URTSPlayerAdvantageComponent::GetSpeedBoostFactor()` | **ADDED** (`BlueprintPure` exposure) | Static per match — read once | Local controller |
| 9 | Control-group chips (`1 ·12 UNITS` …) | `ARTSPlayerController::GetControlGroupMembers(Index)` (validity-pruned); `FRTSControlGroup::Actors` now `BlueprintReadOnly` | **ADDED** | **Poll 4 Hz** (local data, members die without a delegate); `LoadControlGroup(n)` returns whether a configured non-empty group was restored | Local-only by nature |
| 10 | Toast rail (`UNDER ATTACK …`) | `URTSEventFeedSubsystem::OnHudEvent` / per-type streams; `GetRecentEvents()` back-fill | **NEW** | Event | Local aggregation of replicated sources |
| 11 | Screen-edge chevrons & minimap event pings | Same feed: events with `bHasWorldLocation` (`UnderAttack`, `ProductionComplete`, `NodeDepleted`, `UnitLost`, `Ping`) | **NEW** | Event | — |
| 12 | User ping (Alt+click) | `ARTSPlayerController::IssueAlliedPing(WorldLocation, Text)`; direct local-only presentation via `URTSEventFeedSubsystem::EmitPing` | **COMPLETE** server-routed allied pings | `URTSPingComponent::OnPlayerPingReceived`; feed `OnPing` | Reliable client→server→eligible clients; authority team/observer policy + token bucket |
| 13 | Portrait plate | `URTSPortraitComponent::GetPortrait()`; authority mutation through `SetPortrait` | EXISTING getter + **ADDED** transactional mutation/event | Rebind on selection change; `OnPortraitChanged` refreshes live identity changes | Authored asset or replicated runtime asset reference |
| 14 | Unit name / class line | `URTSNameComponent::GetName()`, `URTSDescriptionComponent::GetDescription()`; authority mutation through Boolean setters | EXISTING getters + **ADDED** transactional mutation/events | Rebind on selection change; `OnNameChanged` / `OnDescriptionChanged` refresh live identity changes | Authored data and runtime changes replicate with RepNotify |
| 15 | HP bar (`38 / 160`) | `URTSHealthComponent::GetCurrentHealth()/GetMaximumHealth()` | EXISTING | `OnHealthChanged(Actor, Old, New, DamageCauser)` | `CurrentHealth` replicates; delegate fires on clients |
| 16 | Shield strip (`20 / 20`, above HP) | `URTSHealthComponent::GetCurrentShield()/GetMaximumShield()` (`MaxShield > 0` gates the strip) | EXISTING | `OnShieldChanged(Actor, Old, New)` | Replicates |
| 17 | Energy bar (`87 / 200`, regen ghost-tip) | `URTSEnergyComponent::GetCurrentEnergy()/GetMaximumEnergy()/GetEnergyRegenRate()` | EXISTING | `OnEnergyChanged(Actor, Old, New)` | Replicates |
| 18 | Armor row (`2 MEDIUM` **+1**) | Base: `GetArmor()`, `GetUnitSize()`. Green bonus: `GetArmorUpgradeBonus()`; combat value `GetEffectiveArmor()` | Base EXISTING; split **ADDED** | Re-read on `OnUpgradeLevelChanged` | Upgrade levels replicate |
| 19 | Weapon row (`8` **(+2)** `▼ GROUND 6.5 TLS`) | `URTSAttackComponent::GetAttacks()` → `FRTSAttackData` (`Damage`, `Range` (cm; ÷100 = tiles), `TargetDomain`, `DamageType`, `SplashRadius`, `HitCount`). Green bonus: `GetAttackUpgradeDamageBonus(Index)` | Base EXISTING; bonus split **ADDED** | Re-read on `OnUpgradeLevelChanged` / `OnStanceChanged` (stances can swap the attack set) | Authored data + replicated upgrades |
| 20 | Attack-cooldown sweep | `GetRemainingCooldownForAttack(Index)` (EXISTING), `GetRemainingCooldownFraction(Index)` ∈ [0..1] | Fraction **ADDED** | Animate from the synchronized deadline; rebind on `OnAttackUsed` / `OnCooldownReady` | Server publishes compact deadlines on attack start; smooth and authoritative on relevant clients without per-frame replication |
| 21 | Rank chevrons (kill counter) | `URTSAttackComponent::GetKills()` — credited on the server in the health-component death path to the damage causer's attack component | **NEW** | `OnKillsChanged(Actor, Old, New)` | `Kills` replicates (RepNotify fires the delegate on clients) |
| 22 | Status-tag chips (`CARRYING MINERALS`, SIEGED, …) | `URTSGameplayTagsComponent` → `CurrentTagsChanged(Actor, Tags)`; tag names in `URTSGameplayTagLibrary`. Carried payload: `URTSGathererComponent::GetCarriedResourceType()` (EXISTING) + `GetCarriedResourceAmount()` | Tags EXISTING; carried amount **ADDED** | `CurrentTagsChanged`; `OnCarriedResourcesChanged`; owner-private `OnGatheringSourceChanged` | Tags and payload replicate; active harvesting source is owner-only |
| 23 | Multi-select (`×24 UNITS`, Σ HP, tab rail, tile grid) | `ARTSPlayerController::GetSelectedActors()`; subgroup: `GetSelectedSubgroup/GetSelectedSubgroupActor(s)`, `SelectSubgroup`, `SelectNext/PreviousSubgroup`; sort `URTSSelectableComponent::GetSelectionPriority()`; per-tile HP/shield from each actor's health component (rows 15–16) | EXISTING; **ADDED** assignable rebind triggers (row 24) | Rebind on selection events; tile bars via each `OnHealthChanged` | Selection is local; vitals replicate |
| 24 | Selection rebind plumbing | `ARTSPlayerController::OnSelectionChangedEvent` + `OnSelectedSubgroupChangedEvent` — assignable twins of the `Receive…` BIEs so *any* widget can bind without subclassing the controller | **ADDED** | Event | Local |
| 25 | Command card — orders/stances | Gating `URTSOrderLibrary::CanObeyOrder`; button art on `URTSOrder` CDOs; active stance `URTSStanceComponent::GetActiveStanceName()` + `OnStanceChanged`; world interaction via `BeginOrderTargeting`, `ConfirmTargetingAt`, `CancelTargeting`, and targeting delegates | EXISTING + targeting contract **ADDED** | Rebind on selection; targeting and stance events | Stance replicates with RepNotify; targeting is intentionally local and gameplay requests are server-authoritative |
| 26 | Command card — production buttons + tooltip cost/time | `URTSProductionComponent::GetAvailableProducts()`; per-product CDO `URTSProductionCostComponent::GetResources()/GetProductionTime()`; supply line `URTSSupplyComponent::GetSupplyCost()` (CDO); gate `CheckCanIssueProductionOrder` + `CanAssignProduction` | EXISTING | Rebind on selection; affordability re-check on `OnResourcesChanged` | CDO data — client-safe |
| 27 | Tooltip `REQUIRES {X}` line + lock glyph | `URTSGameplayLibrary::GetMissingRequirementFor(...)` (now `BlueprintPure`) and `GetMissingRequirementNameForUI(...)` → ready-to-render `FText` | **ADDED** (E1) | Re-check on `OnActorOwnerChanged` / production events | Ownership replicates |
| 28 | Command card — abilities (`Q` diamond, energy badge) | `URTSAbilityComponent::GetNumAbilities/GetAbility(Index)` (`FRTSAbility`: name, icon, energy cost, cooldown, range, targeting); gates `CanCast/CanCastAt`; sweep `GetRemainingCooldown(Index)` | EXISTING | `OnAbilityCast`; energy badge via `OnEnergyChanged` | Ability cooldowns replicate |
| 29 | Command card — research (`LV 2→3` badge) | `URTSResearchComponent::GetAvailableResearch()/GetCurrentLevel(Key)/GetNextLevelCost(Key)/CanResearch(Key)`; cancel via `IssueCancelResearchOrder` | EXISTING | `OnResearchStarted/Completed/Canceled/CostRefunded`; progress row 31 | Key/remaining/total replicate |
| 30 | Production progress bar + parallel queue slots + rally | Built into `URTSInfoPanelWidget`: every authored queue appears in a horizontally scrollable group with independent live progress, slots, and indexed cancellation; the primary progress plate follows the first active queue. Data from `GetProgressPercentage/GetRemainingProductionTime/GetCurrentProduction(Queue)`, `GetQueuedProducts(Queue)`, `GetQueueCount/GetCapacityPerQueue`, and `GetRallyPoint()`; owning UI cancels via `IssueCancelProductionOrder(Actor, Queue, Index)` | **COMPLETE** native parallel-queue UI + existing runtime | `OnProductionStarted/ProgressChanged/Finished/Canceled/CostRefunded`, `OnProductQueued`, `OnRallyPointChanged` | Exact queues, timings, rally state, and last product replicate owner-only; mutation is server-routed and revalidated by queue + product index |
| 31 | Research-in-progress bar | `IsResearching()/GetCurrentResearchKey()/GetResearchProgress()` | EXISTING | Interpolate at 10 Hz between reads | Replicates |
| 32 | Research-complete toast | Feed `HUDEVENT_ResearchComplete` — sourced from `URTSUpgradeComponent::OnUpgradeLevelChanged` (**client-safe**; `URTSResearchComponent::OnResearchCompleted` is server-only — never bind client UI to it) | **NEW** (feed) | Event | Replicated upgrades |
| 33 | Construction bar (`CONSTRUCTING — 42%`, `WAITING FOR BUILDER…`) | `URTSConstructionSiteComponent::GetProgressPercentage/GetRemainingConstructionTime/GetState()` (`ERTSConstructionState::NotStarted` etc. labels the bar); builders `GetAssignedBuilders()/GetMaxAssignedBuilders()`; cancel refund `GetRefundFactor()` | EXISTING | `OnConstructionStateChanged`, `OnAssignedBuildersChanged`, plus authoritative `OnConstructionStarted/ProgressChanged/Finished/Canceled/CostRefunded` | State and remaining time replicate; assigned builder identities replicate owner-only |
| 34 | Garrison bay strip | Built into `URTSInfoPanelWidget`: count/capacity, owner-private passenger portraits, click-to-unload and `ALL`; data from `URTSContainerComponent::GetContainedActors()/GetCapacity()/GetOccupancy()`; owning-client actions route through `ARTSPlayerController::IssueUnloadActorOrder/IssueUnloadAllOrder`; passenger side `URTSContainableComponent::GetContainer()` | **COMPLETE** native UI + secure multiplayer contract | `OnActorEntered/OnActorLeft` for owner-private identities, `OnOccupancyChanged` for all relevant clients, `OnContainerChanged` on passengers | Capacity + occupancy replicate to relevant clients; occupant identities replicate owner-only; observers see neutral occupied slots only; controller requests revalidate on authority |
| 35 | Resource-node panel (amount bar, `{n}/cap` gatherers) | `URTSResourceSourceComponent::GetCurrentResources()/GetMaximumResources()/GetResourceType()/GetGathererCapacity()` | EXISTING | `OnResourcesChanged(Source, Old, New)`; `OnCapacityChanged`; depletion via feed `HUDEVENT_NodeDepleted` | Current and maximum resources replicate, including extractor transfers |
| 36 | Bounty floating text | `URTSBountyComponent::OnBountyCollected(Actor, Player, Type, Amount)` | EXISTING | Event | Server-side; feed re-emission local |
| 37 | Unit-lost toast | Feed `HUDEVENT_UnitLost` (from `URTSHealthComponent::OnKilled`, filtered to the local player) | **NEW** (feed) | Event | `OnKilled` fires on clients (health RepNotify hits 0) |
| 38 | Player-eliminated banner-toast | `ARTSGameState::OnPlayerDefeated(ARTSPlayerState*)` — NetMulticast relay of the server-only `ARTSGameMode::NotifyOnPlayerDefeated` (E8); feed `HUDEVENT_PlayerEliminated` | **ADDED** (relay) + **NEW** (feed) | Event | Multicast — reaches all clients |
| 39 | Under-attack alert (toast + ping + chevron) | Feed `HUDEVENT_UnderAttack`: `OnHealthChanged` deltas on owned actors, merged **one alert / 8 s / 4000 UU cluster** (Config tunables); distinct copy for producers (`Status_Permanent_CanProduce`) | **NEW** (feed) | Event; click-to-focus via `FocusCameraOnLocation`, whose Boolean result rejects invalid/non-finite destinations | Health replicates |
| 40 | Error-toast pipe (`ReceiveOnErrorOccurred` had no assignable twin) | `ARTSPlayerController::OnErrorOccurredEvent(Message)` | **ADDED** | Event | Local |
| 41 | Enemy/neutral panel policy | `URTSGameplayLibrary::IsOwnedByLocalPlayer(Actor)` (now `BlueprintPure`) hides command card; `IsFullyVisibleForLocalClient(Actor)` for fog gating; relationship border via `URTSGameplayTagLibrary::GetActorRelationshipTags` | **ADDED** exposure / EXISTING | Rebind on selection | Never render non-replicated enemy state (energy, cooldowns) |
| 42 | Match end banner | `ARTSPlayerController::ReceiveOnGameHasEnded(bIsWinner)`; `Surrender()` returns whether the authoritative/local request was accepted for submission | EXISTING | Event | Client RPC |
| 43 | Hover nameplate | `URTSSelectableComponent::OnHovered/OnUnhovered`, `ARTSPlayerController::GetHoveredActor()` | EXISTING | Event | Local |
| 44 | Minimap layers (terrain/fog/dots/frustum) | `URTSMinimapWidget` + `ARTSMinimapVolume` + `ARTSFogOfWarActor::GetFogOfWarTexture()` | EXISTING (styling: other work stream) | NativePaint | Vision is per-player |

---

## 2. The event feed — `URTSEventFeedSubsystem` (`Public/Feedback/`)

`ULocalPlayerSubsystem`. Get it in Blueprint: **Get Local Player Subsystem → RTS Event Feed
Subsystem** (from any widget or the player controller). It subscribes once to every delegate in
the tables above, applies the merge/throttle rules, and re-emits one typed stream.

**Event struct `FRTSHudEvent`** (`Feedback/RTSHudEvent.h`): stable local `EventId`, `EventType`, `WorldLocation` +
`bHasWorldLocation`, `Source` (actor), `SubjectClass`, `ResourceType`, `UpgradeKey`, `Text`
(ready-to-render line), `Value`, `Timestamp`, `MergedCount`.

**Event types `ERTSHudEventType`**: `UnderAttack`, `ProductionComplete`, `ResearchComplete`,
`SupplyBlocked`, `UnitLost`, `PlayerEliminated`, `Ping`, `NodeDepleted`, `CostRefunded`.

**Delegates**: `OnHudEvent` creates a new presentation event. `OnHudEventUpdated` refreshes an
existing event in place—match `EventId` and do not replay toast sounds, animations, or markers.
Under-attack burst merges use this channel to update `MergedCount` without producing duplicate
alerts. Per-type create conveniences: `OnUnderAttack`,
`OnProductionComplete`, `OnResearchComplete`, `OnSupplyBlocked`, `OnUnitLost`,
`OnPlayerEliminated`, `OnPing`.

**Functions**: local-only `EmitPing(WorldLocation, Text, Sender)` ·
`UpdateEventMergedCount(EventId, NewCount)` · `GetIncomePerMinute(ResourceType)` ·
`GetRecentEvents()` (back-fill for late-constructed widgets, including current merged counts).

For multiplayer user pings, call `ARTSPlayerController::IssueAlliedPing`. Its auto-created
`URTSPingComponent` performs authority routing, then injects the received event into each eligible
local player's feed. See `TEAM_PINGS.md`.

**Config tunables** (`[/Script/RealTimeStrategy.RTSEventFeedSubsystem]` in `DefaultGame.ini`):
`UnderAttackThrottleSeconds=8`, `UnderAttackClusterDistance=4000`,
`SupplyBlockedThrottleSeconds=10`, `IncomeWindowSeconds=60`, `MaxRecentEvents=32`.

**Cost**: event-driven only, no per-frame work; every handler early-outs when nothing is bound.
An unbound feed costs one delegate dispatch per gameplay event.

---

## 3. Worked examples (Blueprint terms)

### 3.1 Resource counter in 4 nodes
In your HUD widget's **Event Construct**:
1. **Get Owning Player** → **Get Component By Class** (`RTSPlayerResourcesComponent`).
2. **Bind Event to On Resources Changed** (custom event `HandleResources`).
3. In `HandleResources`: **Set Text** on your TextBlock from `New Resource Amount` (filter on
   `Resource Type` == your chip's type, or just re-read `GetResources`).
4. Call `HandleResources` once manually after binding (initial value; `GetResources(Type)`).

Income sub-chip: same handler, add **Get Local Player Subsystem (RTS Event Feed Subsystem)** →
`GetIncomePerMinute(Type)` → format `+{0}/min`. Spend-flash: only when `bSyncedFromServer` is
**false** (the server echo repeats the change).

### 3.2 Under-attack minimap ping
In your minimap widget's **Event Construct**:
1. **Get Local Player Subsystem** (`RTSEventFeedSubsystem`).
2. **Bind Event to On Under Attack** (custom event `HandleUnderAttack`).
3. In `HandleUnderAttack`: `HudEvent.World Location` → your world→minimap UV transform → spawn a
   ping ring; also compare against the camera frustum and spawn a screen-edge chevron when
   off-screen. `HudEvent.Text` is the toast line ("YOUR BASE IS UNDER ATTACK" for producers).
4. Click-to-focus: store `World Location`; on click call `FocusCameraOnLocation` (player controller).

The 8 s / 4000 UU merge is already applied — you receive at most one event per cluster per window.

### 3.3 Production progress bar
When the selection changes (**Bind Event to On Selection Changed Event** on the RTS player
controller — no controller subclass needed):
1. On the selected building: **Get Component By Class** (`RTSProductionComponent`).
2. **Bind Event to On Production Progress Changed** → **Set Percent** on the ProgressBar from
   `Progress Percentage` (0..1). Bind **On Production Finished** to clear/advance.
3. Label: `GetCurrentProduction(Queue)` → class display name (or its `RTSPortraitComponent` CDO
   portrait via `URTSGameplayLibrary::FindDefaultComponentByClass`); time
   `GetRemainingProductionTime(Queue)` formatted mm:ss.
4. Queue slots: `GetQueuedProducts(Queue)` → one slot per entry; slot click →
   `CancelProduction(Queue, Index)`; refunds arrive on `OnProductionCostRefunded` (and as a
   `CostRefunded` feed toast).

Queues replicate — this exact wiring works on remote clients.

---

## 4. Polling exceptions (and why)

| Data | Rate | Why polling is correct |
|---|---|---|
| Supply used/cap | 4 Hz | Sum over replicated actors; no aggregate delegate exists (and a per-actor one would be noisier than the poll) |
| Control groups | 4 Hz | Purely local player data; members die without any group-scoped event |
| Match clock | 1 Hz | It's a clock |
| Cooldown sweeps / progress interpolation | UI tick, local | Animation smoothing between authoritative events |

Everything else in this document is event-driven.
