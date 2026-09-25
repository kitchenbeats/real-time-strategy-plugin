# UI Data API

**Everything you need to bind your own HUD to game data.** Every getter below is `BlueprintPure` and every
state-change event is a `BlueprintAssignable` delegate. UI-specific additions use the Blueprint
category **`RTS|UI Data`**; decomposed selection and order APIs retain their owning
**`RTS|Selection`**, **`RTS|Orders`**, and **`RTS|Player`** categories. Widgets never poll per-frame
except where this document explicitly says "poll" (local-only data with no delegate — supply,
control groups).

The presentation contract is `URTSHudStyle`. `URTSEventFeedDemoWidget` under `Public/Feedback/` is
the native reference for building a Blueprint or C++ presentation layer without a project-specific
asset-generation script.

Two player-facing components have their own getters: obtain local selection through
`ARTSPlayerController::GetSelection()` (`URTSSelectionComponent`) and selected-unit submission through
`ARTSPlayerController::GetOrderSubmission()` (`URTSOrderSubmissionComponent`). The controller still
publishes the Blueprint-assignable selection and error events listed below.

---

## 0. Presentation policies and supported limits

Everything in the HUD element matrix (section 1) is backed by a real getter or delegate. The remaining limits are
intentional product policies rather than missing data:

| Policy | Supported behavior |
|---|---|---|
| **Team swatch color** | `URTSHudStyle::TeamColors` resolves the replicated team index. Projects can replace the palette without changing gameplay. |
| **Rank chevron tiers** | `URTSHudStyle::RankKillThresholds` maps replicated kills to presentation tiers (default 1/3/6). |
| **Allied player pings** | `ARTSPlayerController::IssueAlliedPing` uses the replicated `RTSPingComponent` for validated, rate-limited, authority-filtered delivery. Sender and recipient policies are Blueprint/C++ overridable. |
| **Sim-speed of other human players** | Remote controllers intentionally do not exist on clients. The match-strip marker covers the local player and server-side AI/observer data without replicating private controller state. |

---

## 1. HUD element → data source matrix

Everything in this table is available to Blueprint and C++. The **State** column is historical: it
records whether the data was already exposed (**EXISTING**), exposed on an existing system
(**ADDED**), or newly tracked (**NEW**) when the HUD data API was built.

| # | HUD element | Data source (component → member) | State | Event / refresh | Replication |
|---|---|---|---|---|---|
| 1 | Resource counter values (`1,240` / `655`) | `URTSPlayerResourcesComponent` (on the **controller**) → `GetResources(Type)`; chip set from `GetResourceTypes()` | EXISTING; `GetResourceTypes` **ADDED** (was C++-only) | `OnResourceCatalogChanged` rebuilds chips; `OnResourcesChanged(Type, Old, New, bSyncedFromServer)` updates exact balances | Catalog and balances replicate owner-only; `bSyncedFromServer` distinguishes server echo (suppress spend-flash on it) |
| 2 | Income sub-chips (`+824/min`) | `URTSEventFeedSubsystem::GetIncomePerMinute(Type)` — 60 s rolling window of positive stock deltas | **NEW** | Recompute on each `OnResourcesChanged`, or on a slow (1 Hz) tick | Local derivation of replicated stock |
| 3 | Supply chip (`53/58`, NEAR CAP) | `URTSGameplayLibrary::GetPlayerSupply(WCO, Controller, Used, Cap)`; per-actor `URTSSupplyComponent::GetSupplyCost/GetSupplyProvided` | EXISTING | **Poll 4 Hz** (sums replicated actors; no delegate) | Client-safe |
| 4 | Supply-blocked flash / toast | `ARTSPlayerController::OnSupplyBlockedEvent(ProductClass)`; feed `HUDEVENT_SupplyBlocked` (10 s throttle) | **ADDED** (delegate + client-side supply check in `CheckCanIssueProductionOrder`) | Event | Local (fires where the order was issued) |
| 5 | Upgrade pips (`W2 A1 S0`) | `URTSUpgradeComponent` (on the controller) → `GetUpgradeLevel(Key)`; per-actor static `GetUpgradeLevelForActor(Actor, Key)` | EXISTING | `OnUpgradeLevelChanged(Key, Old, New)` (EXISTING) | `Upgrades` replicates; delegate fires on clients via RepNotify |
| 6 | Match clock (`12:47`) | `ARTSGameState::GetMatchTimeSeconds()` | **NEW** (replicated `MatchStartServerTime`) | Poll 1 Hz (it's a clock) | Server time — all clients agree |
| 7 | Player name / team swatch | `APlayerState::GetPlayerName()`; `ARTSPlayerState::GetTeam()` → `ARTSTeamInfo::GetTeamIndex()` | EXISTING | `ReceiveOnTeamChanged` | Replicates |
| 8 | Sim-speed marker (`×1.5`) | `URTSPlayerAdvantageComponent::GetSpeedBoostFactor()` | **ADDED** (`BlueprintPure` exposure) | Static per match — read once | Local controller |
| 9 | Control-group chips (`1 ·12 UNITS` …) | `ARTSPlayerController::GetSelection()` → `URTSSelectionComponent::GetControlGroupMembers(Index)` (validity-pruned); `FRTSControlGroup::Actors` is `BlueprintReadOnly` | **ADDED** | **Poll 4 Hz** (local data, members die without a delegate); `GetSelection()->LoadControlGroup(n)` returns whether a configured non-empty group was restored | Local-only by nature |
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
| 23 | Multi-select (`×24 UNITS`, Σ HP, tab rail, tile grid) | `ARTSPlayerController::GetSelection()` → `URTSSelectionComponent::GetSelectedActors()`; subgroup: `GetSelectedSubgroup`, `GetSelectedSubgroupActor`, `GetSelectedSubgroupActors`, `SelectSubgroup`, `SelectNextSubgroup`, `SelectPreviousSubgroup`; sort `URTSSelectableComponent::GetSelectionPriority()`; per-tile HP/shield from each actor's health component (rows 15–16) | EXISTING; **ADDED** assignable rebind triggers (row 24) | Rebind on selection events; tile bars via each `OnHealthChanged` | Selection is local; vitals replicate |
| 24 | Selection rebind plumbing | `ARTSPlayerController::OnSelectionChangedEvent` + `OnSelectedSubgroupChangedEvent` — the controller republishes the selection component's changes through assignable twins of the `Receive…` BIEs so *any* widget can bind without subclassing the controller | **ADDED** | Event | Local |
| 25 | Command card — orders/stances | Gating `URTSOrderLibrary::CanObeyOrder`; selected-order submission through `GetOrderSubmission()->IssueOrderToSelectedActors`; button art on `URTSOrder` CDOs; active stance `URTSStanceComponent::GetActiveStanceName()` + `OnStanceChanged`; targeting interaction via controller `BeginOrderTargeting`, `ConfirmTargetingAt`, `CancelTargeting`, and targeting delegates | EXISTING + targeting contract **ADDED** | Rebind on selection; targeting and stance events | Stance replicates with RepNotify; targeting is intentionally local; selected orders use the replicated client-owned submission component and remain server-authoritative |
| 26 | Command card — production buttons + tooltip cost/time | `URTSProductionComponent::GetAvailableProducts()`; per-product CDO `URTSProductionCostComponent::GetResources()/GetProductionTime()`; supply line `URTSSupplyComponent::GetSupplyCost()` (CDO); gate `CheckCanIssueProductionOrder` + `CanAssignProduction` | EXISTING | Rebind on selection; affordability re-check on `OnResourcesChanged` | CDO data — client-safe |
| 27 | Tooltip `REQUIRES {X}` line + lock glyph | `URTSGameplayLibrary::GetMissingRequirementFor(...)` (now `BlueprintPure`) and `GetMissingRequirementNameForUI(...)` → ready-to-render `FText` | **ADDED** (E1) | Re-check on `OnActorOwnerChanged` / production events | Ownership replicates |
| 28 | Command card — abilities (`Q` diamond, energy badge) | `URTSAbilityComponent::GetNumAbilities/GetAbility(Index)` (`FRTSAbility`: name, icon, energy cost, cooldown, range, targeting); gates `CanCast/CanCastAt`; sweep `GetRemainingCooldown(Index)` | EXISTING | `OnAbilityCast`; energy badge via `OnEnergyChanged` | Ability cooldowns replicate |
| 29 | Command card — research (`LV 2→3` badge) | `URTSResearchComponent::GetAvailableResearch()/GetCurrentLevel(Key)/GetNextLevelCost(Key)/CanResearch(Key)`; cancel via `IssueCancelResearchOrder` | EXISTING | `OnResearchStarted/Completed/Canceled/CostRefunded`; progress row 31 | Key/remaining/total replicate to owner only; names/descriptions are authored catalog data |
| 30 | Production progress bar + parallel queue slots + rally | Built into `URTSInfoPanelWidget`: every authored queue appears in a horizontally scrollable group with independent live progress, slots, and indexed cancellation; the primary progress plate follows the first active queue. Data from `GetProgressPercentage/GetRemainingProductionTime/GetCurrentProduction(Queue)`, `GetQueuedProducts(Queue)`, `GetQueueCount/GetCapacityPerQueue`, and `GetRallyPoint()`; owning UI cancels via `IssueCancelProductionOrder(Actor, Queue, Index)` | **COMPLETE** native parallel-queue UI + existing runtime | `OnProductionStarted/ProgressChanged/Finished/Canceled/CostRefunded`, `OnProductQueued`, `OnRallyPointChanged` | Exact queues, timings, rally state, and last product replicate owner-only; mutation is server-routed and revalidated by queue + product index |
| 31 | Research-in-progress bar | `IsResearching()/GetCurrentResearchKey()/GetResearchProgress()` | EXISTING | Interpolate at 10 Hz between reads | Replicates |
| 32 | Research-complete toast | Feed `HUDEVENT_ResearchComplete` — sourced from `URTSUpgradeComponent::OnUpgradeLevelChanged` (**client-safe**; `URTSResearchComponent::OnResearchCompleted` is server-only — never bind client UI to it) | **NEW** (feed) | Event | Replicated upgrades |
| 33 | Construction bar (`CONSTRUCTING — 42%`, `WAITING FOR BUILDER…`) | `URTSConstructionSiteComponent::GetProgressPercentage/GetRemainingConstructionTime/GetState()` (`ERTSConstructionState::NotStarted` etc. labels the bar); builders `GetAssignedBuilders()/GetMaxAssignedBuilders()`; cancel refund `GetRefundFactor()` | EXISTING | `OnConstructionStateChanged`, `OnAssignedBuildersChanged`, plus authoritative `OnConstructionStarted/ProgressChanged/Finished/Canceled/CostRefunded` | State and remaining time replicate; assigned builder identities replicate owner-only |
| 34 | Garrison bay strip | Built into `URTSInfoPanelWidget`: count/capacity, owner-private passenger portraits, click-to-unload and `ALL`; data from `URTSContainerComponent::GetContainedActors()/GetCapacity()/GetOccupancy()`; owning-client actions route through `ARTSPlayerController::IssueUnloadActorOrder/IssueUnloadAllOrder`; passenger side `URTSContainableComponent::GetContainer()` | **COMPLETE** native UI + secure multiplayer contract | `OnActorEntered/OnActorLeft` for owner-private identities, `OnOccupancyChanged` for all relevant clients, `OnContainerChanged` on passengers | Capacity + occupancy replicate to relevant clients; occupant identities replicate owner-only; observers see neutral occupied slots only; controller requests revalidate on authority |
| 35 | Resource-node panel (amount bar, `{n}/cap` gatherers) | `URTSResourceSourceComponent::GetCurrentResources()/GetMaximumResources()/GetResourceType()/GetGathererCapacity()` | EXISTING | `OnResourcesChanged(Source, Old, New)`; `OnCapacityChanged`; depletion via feed `HUDEVENT_NodeDepleted` | Current and maximum resources replicate, including extractor transfers |
| 36 | Bounty floating text | `URTSBountyComponent::OnBountyCollected(Actor, Player, Type, Amount)` | EXISTING | Event | Server-side; feed re-emission local |
| 37 | Unit-lost toast | Feed `HUDEVENT_UnitLost` (from `URTSHealthComponent::OnKilled`, filtered to the local player) | **NEW** (feed) | Event | `OnKilled` fires on clients (health RepNotify hits 0) |
| 38 | Player-eliminated banner-toast | `ARTSGameState::OnPlayerDefeated(ARTSPlayerState*)` — NetMulticast relay of the server-only `ARTSGameMode::NotifyOnPlayerDefeated` (E8); feed `HUDEVENT_PlayerEliminated` | **ADDED** (relay) + **NEW** (feed) | Event | Multicast — reaches all clients |
| 39 | Under-attack alert (toast + ping + chevron) | Feed `HUDEVENT_UnderAttack`: `OnHealthChanged` deltas on owned actors, merged **one alert / 8 s / 4000 UU cluster** (Config tunables); distinct copy for producers (`Status_Permanent_CanProduce`) | **NEW** (feed) | Event; click-to-focus via `GetCameraControl()->FocusCameraOnLocation`, whose Boolean result rejects invalid/non-finite destinations | Health replicates |
| 40 | Error-toast pipe (`ReceiveOnErrorOccurred` had no assignable twin) | `ARTSPlayerController::OnErrorOccurredEvent(Message)` | **ADDED** | Event | Local |
| 41 | Enemy/neutral panel policy | `URTSGameplayLibrary::IsOwnedByLocalPlayer(Actor)` (now `BlueprintPure`) hides command card; `IsFullyVisibleForLocalClient(Actor)` for fog gating; relationship border via `URTSGameplayTagLibrary::GetActorRelationshipTags` | **ADDED** exposure / EXISTING | Rebind on selection | Never render non-replicated enemy state (energy, cooldowns) |
| 42 | Match end banner | `ARTSPlayerController::ReceiveOnGameHasEnded(bIsWinner)`; `Surrender()` returns whether the authoritative/local request was accepted for submission | EXISTING | Event | Client RPC |
| 43 | Hover nameplate | `URTSSelectableComponent::OnHovered/OnUnhovered`, `ARTSPlayerController::GetHoveredActor()` | EXISTING | Event | Local |
| 44 | Minimap layers (terrain/fog/dots/frustum) | `URTSMinimapWidget` + `ARTSMinimapVolume` + `ARTSFogOfWarActor::GetFogOfWarTexture()` | EXISTING (styling: other work stream) | NativePaint | Vision is per-player |

---

### Accepted order-plan binding

On the owning RTS player controller, use **Get Order Submission** → bind
**On Accepted Order Plan Changed** → **Get Accepted Order Plan**. The Blueprint-assignable event
has no payload. Read once after binding, refresh from the event, and unbind on widget teardown.
The component automatically inspects one selected owned pawn; do not add another submission
component or send gameplay orders to refresh this view.

Check `FocusedPawn` and clear the view when it is absent or differs from your displayed selection.
An empty result can mean awaiting authority or stale/unsupported focus, rather than an idle unit.
For a valid snapshot, display `CurrentOrder`, ordered `QueuedOrders`, and
`QueuedCount / QueueCapacity`; show remaining commands when your display limit or `bTruncated`
omits steps. The native fallback info panel and world cues display at most 12 commands; transport
carries at most 64 pending steps, independent of the gameplay queue's default capacity of 16.
The native panel reserves its progress area for production/construction when those are displayed.

This is owner-only accepted state. Draw `Destination` only when `bHasDestination` is true;
actor-target steps omit target references and redact positions outside the owner's visibility,
except friendly targets. Break lines at unavailable destinations and label them as command
sequence, not navigation paths. Queue-full feedback uses the existing controller
`OnErrorOccurredEvent`. See [ORDERS.md](ORDERS.md#inspect-accepted-command-plans) for field,
privacy, acknowledgement and rejection semantics.

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
4. Click-to-focus: store `World Location`; on click call
   `GetCameraControl()->FocusCameraOnLocation` on the RTS player controller.

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

### 3.4 Customize the skirmish setup screen

The included setup screen is `RTSSkirmishSetupWidget`. It appears for the local host when the
active skirmish definition has **Require Player Setup** enabled. The installed complete starter
already enables this setting. Faction names, descriptions, crests, and starting classes come from
the definition's faction catalog.

Create a Widget Blueprint with **RTSSkirmishSetupWidget** as its parent. Arrange these controls in
the Designer, preserving their exact names and types:

| Widget name | Designer type | Purpose |
|---|---|---|
| `PlayerFaction` | Combo Box (String) | The host's faction choice |
| `OpponentCount` | Combo Box (String) | Number of AI opponents |
| `OpponentRows` | Vertical Box | Container populated with opponent faction choices |
| `SetupError` | Text Block | Backend validation or startup error |
| `StartMatchButton` | Button | Validates the lineup and starts the authoritative match |
| `FactionDescription` | Text Block, optional | Selected faction description |
| `FactionCrest` | Image, optional | Selected faction crest; hidden when none is assigned |

The five required controls use optional Unreal widget bindings so the native class also works
without a Widget Blueprint. At runtime the screen checks their names and types. An incomplete
layout produces a diagnostic and falls back to the full native interface. Omitted description and
crest controls are supported. With one faction, the player faction control is disabled, keyboard
focus starts on opponent count, and redundant opponent faction rows are omitted.

Opponent choices come from `RTSSkirmishGameMode::GetSupportedOpponentCounts()`: a read-only Blueprint
query that respects the current map's coverage, base spacing, resources, and required human slots.
Use the returned values when building your own controls; they need not start at one or be contiguous.
The native screen preserves a saved supported choice and disables Start with an explanation when
no setup fits. Startup still validates the chosen lineup against the current battlefield.

The native parent populates options and owns the start action; no Blueprint click graph is needed.
It displays the backend's validation reason and leaves the setup available after a rejected action.
Successful start hides setup, restores the gameplay console and Game-and-UI input, and keeps the
RTS camera. The match clock begins when gameplay starts, excluding time spent choosing a lineup.

To use your layout, create a Blueprint child of **RTSHUD** and set **Skirmish Setup Widget Class**
to your Widget Blueprint. Assign that HUD class in your own GameMode. For a generated
starter, create a child of its generated GameMode outside the generated folder and select that child
in your own copy of the map (also outside the generated folder) under
**World Settings > GameMode Override**. A custom skirmish definition
can also select the HUD through `Definition.HUDClass`; that explicit value takes precedence over
the GameMode default. Keep custom widgets, HUDs, GameModes, and maps outside the generation manifest's
owned outputs so regenerating gameplay preserves your UI work.

The default layout uses the project's `RTSHudStyle`. A Designer layout keeps its authored control
styles; dynamically populated opponent rows use the same shared HUD style. Standalone setup is a local host workflow. In a hosted free-for-all, the same screen displays the
replicated lobby described below; remote players can change only their own faction and readiness.


### Multiplayer roster and readiness

After hosting, setup becomes a **free-for-all lobby**. The opponent count controls total bases after
the first base. Connected participants occupy stable bases; the remaining bases belong to AI. Each
player selects their own faction. The host configures AI factions and the number of available bases;
a base occupied by another human cannot be removed until that player chooses **Spectate this match**.
Spectators can request an available base while setup is open.

Remote participants click **Ready** before the host can start. Changing the lineup or connected
participants clears remote readiness and advances `LineupRevision`. The owning controller includes
the displayed revision in its request, so a delayed Ready cannot approve a newer, unseen lineup.
The server validates readiness even when Start is called from
Blueprint or C++. Joining a running match enters as a spectator. A multiplayer rematch returns everyone
to setup for renewed readiness, keeping participant bases and spectator roles. Single-player rematch
starts another match immediately after cleanup.

Customer interfaces read `RTSGameState::GetSkirmishLobbyState()` and bind
`OnSkirmishLobbyChanged`. The snapshot contains the authoritative faction catalog, faction IDs by
base, supported opponent counts, connected player states, base assignments, readiness, and the
reason Start is blocked. `bEnabled` identifies network setup and `bOpen` identifies its editable phase.
A player entry with `BaseIndex == -1` is a spectator. Display player names through the entry's
PlayerState. Call `RTSPlayerController::RequestSkirmishLobbySelection(FactionId, bReady, bSpectator)`
on the owning local controller. Rejections arrive through that controller's existing
`OnErrorOccurredEvent`; requests cannot choose another player's identity or submit asset paths.

The native layout reuses `OpponentRows` for the roster, spectator action, and AI faction controls.
`StartMatchButton` becomes Ready/Not ready for remote participants and remains Start for the host.
There is no team selector: this mode resolves one winner in a free-for-all.

### 3.5 Game menu, settings, and multiplayer entry

Press **Esc** or click **Menu · Esc** during setup or play. If an order target or building preview
is active, the first Esc cancels that preview. The menu provides Resume, display/audio/control settings,
confirmed Surrender, Return to setup, and Quit. Hosting and joining are available from local setup.
Enter the host's address and port to join; the host opens a port before starting the match. A pending
connection exposes **Cancel connection**. Hosted return/quit confirmations explain that all connected
players will leave the hosted game.

Standalone play pauses while the menu is open. Multiplayer continues, and the menu says so. Closing
restores the menu's own input and pause changes while preserving other systems' suppression and the
match-end pause. Surrender uses the existing authoritative player transaction and permits observing
the result. Display settings use Unreal's `GameUserSettings`; master volume is stored in the same
user settings INI under `/Script/RealTimeStrategy.PlayerAudio`. Cancel discards the volume preview
and unapplied display choices. Camera controls stage an edge-scrolling toggle and scroll-speed
multiplier (0.25–3×); Restore camera defaults stages enabled/1×. Apply verifies both preferences
on disk in `/Script/RealTimeStrategy.PlayerControls` before updating every cached local player in
the Game Instance. A failed save keeps the settings panel open. The multiplier preserves authored
camera speed and zoom scaling; disabling edge scrolling leaves keyboard pan available.

For a custom button, call `RTSPlayerController::ToggleGameMenu()`; `RTSHUD::IsGameMenuOpen()` reports
its visibility. To replace the layout, create a Widget Blueprint child of `RTSGameMenuWidget` and
assign it to your HUD's **Game Menu Widget Class**. Preserve these exact names and Designer types:

| Widget names | Designer type |
|---|---|
| `HomePanel`, `SettingsPanel`, `ConfirmationPanel`, `ConnectionPanel` | Vertical Box |
| `MenuStatus`, `ConfirmationText`, `VolumeLabel` | Text Block |
| `ResumeButton`, `SettingsButton`, `SurrenderButton`, `ReturnButton`, `QuitButton` | Button |
| `HostButton`, `JoinButton`, `ApplySettingsButton`, `BackButton` | Button |
| `ConfirmButton`, `CancelConfirmationButton` | Button |
| `WindowMode`, `GraphicsQuality`, `FrameLimit` | Combo Box (String) |
| `VSync` | Check Box |
| `MasterVolume` | Slider |
| `HostPort`, `JoinAddress` | Editable Text Box |

The native class binds these controls and supplies behavior without Blueprint click graphs. Put a
Text Block directly inside Return and Quit buttons to receive the contextual host/cancel captions.
Keep connection controls inside `ConnectionPanel` and normal actions inside `HomePanel`. Missing or
mistyped required controls produce a diagnostic and fall back to the complete native layout. The
native layout uses `RTSHudStyle`; an authored layout retains its own styles. Call `CloseMenu()` when
closing through a custom action so pause, focus, and input ownership are restored.

Camera controls are optional in older custom layouts. Add `EdgeScrolling` (Check Box), `ScrollSpeed`
(Slider), `ScrollSpeedLabel` and `CameraSettingsHint` (Text Blocks), and
`RestoreCameraDefaultsButton` (Button) to use the native staged flow. Custom controls can instead
get the owning LocalPlayer’s `RTSPlayerControlsSubsystem`, read `IsEdgeScrollingEnabled` and
`GetScrollSpeedMultiplier`, and call `ApplyCameraPreferences` with error handling. Bind
`OnCameraPreferencesChanged` to update other views after a verified save. This camera-only API does
not remap keys; the combined keyboard/camera API below applies runtime profile overrides without
rewriting the project's authored input assets.

To include the native keyboard panel in a custom layout, add `KeyboardBindings` (Vertical Box),
`KeyboardBindingsHint` (Text Block), and `RestoreKeyboardDefaultsButton` (Button) inside Settings.
The native menu creates the key selectors inside that Vertical Box and uses the existing Apply button.

Keyboard controls stage nine essential actions: add selection/queue orders, Stop, Move, Attack,
Hold, Patrol, Gather, center selection, and follow selection. They read the owning controller's
actual Enhanced Input context and active profile. The menu accepts single keyboard keys, preserves
authored default overlaps, rejects new conflicts, and prevents clearing an essential binding.
Escape cancels key capture; Cancel discards the staged batch. Restore keyboard defaults stages only
these nine authored defaults. Apply verifies the native saved profile and rolls back the keyboard
changes if either the keyboard or camera save fails.

Custom controls can capture `GetKeyboardControls`, preserve its `ContextId` and row identities,
edit each row's `Key`, and submit the complete snapshot to `ApplyControlPreferences`. Refresh the
snapshot after a successful save or a reported profile change. `bEditable` and `Status` explain
unsupported or ambiguous slots; projects with custom persistence retain their own controls flow.
Bind `OnKeyboardBindingsChanged` to refresh hints after the owning player's effective mappings
rebuild. Command-card hints use those effective mappings rather than the bundled default keys.

Custom connection screens obtain `RTSGameSessionSubsystem` from the Game Instance (C++:
`GameInstance->GetSubsystem<URTSGameSessionSubsystem>()`). Bind `OnSessionChanged` and display
`GetStatusText`, `GetLastError`, and `IsTravelPending`. `HostCurrentMap` starts a listener from saved
local skirmish setup; `JoinAddress` validates and submits a connection request. A true return from
`JoinAddress` means the request was accepted, not that the connection has completed. The subsystem
survives travel and publishes subsequent success or failure. `ReturnToSetup` returns to the saved
local setup map and also cancels a pending connection. These methods report immediate rejection
through their output error text; retain the status delegate for asynchronous failures.

---

## 4. Polling exceptions (and why)

| Data | Rate | Why polling is correct |
|---|---|---|
| Supply used/cap | 4 Hz | Sum over replicated actors; no aggregate delegate exists (and a per-actor one would be noisier than the poll) |
| Control groups | 4 Hz | Purely local player data; members die without any group-scoped event |
| Match clock | 1 Hz | It's a clock |
| Cooldown sweeps / progress interpolation | UI tick, local | Animation smoothing between authoritative events |

Everything else in this document is event-driven.

## Ownership accents and idle overview

The bundled starter uses one absolute team palette across world accents, minimap dots and lobby
swatches. Edit `URTSHudStyle::TeamColors` to replace the palette; the native defaults provide 16
entries. Colors repeat if a custom palette has fewer entries than the configured teams.
`URTSMinimapWidget::UnitColorMode` defaults to **Relationships** for existing customer widgets:
own actors use the own brush, allied actors use `AllyGreen`, enemies use the enemy brush, and
unowned resources use the neutral brush. The bundled minimap selects **TeamPalette**. Both modes
retain visibility filtering and the local under-attack blink.

The native minimap draws mobile units as compact squares, buildings as larger squares, and
resource-drain bases as outlined squares with a center. Currently visible mineral and gas nodes
use neutral diamonds; extractor buildings retain their player color even when their pool is empty.
These solid marker shapes scale from the existing owner/neutral brush dimensions. A texture or
material assigned to a brush keeps its authored image and size instead. Marker positions are
centered on the world-to-minimap coordinate; `OnDrawUnit` still receives that center for custom
drawing, after native visibility and dead-actor filtering. The existing draw-units toggle also
controls native resource markers, allowing a Blueprint to supply its own eligible markers.

Resource nodes have no live minimap marker outside current sight, including explored terrain.
Changing their amount, depletion, destruction or ownership under fog cannot update that absent
marker. Enemy buildings use their existing frozen last-seen ghosts; the minimap never queries a
ghost's hidden source. Friendly forces remain visible according to normal team visibility, while
local hide reasons still apply. Terrain and fog paint below admitted markers, keeping information
the player may see readable. `ARTSMinimapVolume::MinimapImage` remains the terrain-image override;
without an image, `EmptyBackgroundColor` supplies a plain arena surface with no invented terrain
or baked actor locations. Customer terrain images should contain terrain only, without hidden
units or undiscovered resources.

For custom art, add `URTSTeamColorComponent` to the gameplay actor alongside `URTSOwnerComponent`.
Add a binding for each accent channel:

| Binding field | Set it to |
|---|---|
| `MeshComponentName` | The Blueprint variable name of a static or skeletal mesh component on this actor. |
| `MaterialSlotName` | The mesh's named material slot, or None for slot zero. |
| `ColorParameterName` | An existing vector parameter in that material, such as `TeamColor`. The bundled placeholder master uses `Tint`. |

Bindings are explicit opt-in: an empty list changes nothing. The component creates private dynamic
material instances, preserves other material parameters, and never changes the shared source asset.
Ownership and player-team changes update through events. An unresolved initial team index stays
neutral and is checked at 0.1-second intervals only until the immutable index arrives. Dedicated
servers do not create presentation materials. After swapping meshes, materials, bindings, or the
active style at runtime, call `RefreshTeamColor()`. It returns false for invalid channels and logs
the offending binding; more than 64 bindings rejects the entire refresh without changing currently
applied materials. Clearing bindings or removing the component restores its original materials,
provided another system has not replaced them in the meantime.

Generated placeholders tint the unit head or building cap, retaining the body's role color and
silhouette. Switching the content definition to assigned art removes the generator-owned
`GreyboxTeamColor`; independently named customer components remain yours. Bind the replacement
art explicitly. Fog ghosts copy dynamic parameter values and capture the team index at the moment
of last sight: later source ownership, team, or material changes do not update the saved appearance.

The idle info panel polls only the local player's owned actors at 4 Hz, separating live units from
buildings and displaying an observer-specific inspection message when appropriate. Set
`URTSHudStyle::MatchObjective` to your game's objective; an empty value omits it. The bundled starter
uses “Destroy opposing bases. Keep yours standing.” This is presentation text, not a victory-rule
configuration.

### Game name and window title

Use **Tools > Real-Time Strategy > Rename Game** to set the product name. The action saves the
existing Unreal **Project Settings > Description > Project Name** and derives the standalone window
title from that same name. Native setup and Escape-menu screens read this identity; custom Widget
Blueprints can use **Get Game Display Name** (`URTSGameIdentityLibrary`). Reopen an already open
screen or launch a new standalone window after renaming.

Editor Utility Blueprints and Python can call **Set Game Display Name** on
`URTSContentSetEditorLibrary`. Names are trimmed, limited to 80 characters, and reject control
characters. The editor action validates and verifies both persisted settings before publishing the
new live identity; errors explain an invalid name or unwritable project configuration. Unrelated
project identifiers and INI array operations remain intact. The window title treats braces and other
format delimiters in your name literally. Non-shipping Unreal builds may still append their normal
platform/build information.

A Content Set's display name labels that ruleset asset; faction, unit, building, and resource display
names remain their own editable definitions. This action does not rename the `.uproject`, executable,
application bundle, or store listing. Configure those packaging identities separately before release.
