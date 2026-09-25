# Supported C++ API Surface

This document identifies the native entry points the Real-Time Strategy plugin supports for game
modules. Include headers by their domain path, add `RealTimeStrategy` to the consuming module's
dependencies, and keep project-owned subclasses in the project's `Source` tree.

Public placement makes a type available to compile because exported signatures may require it. The
customer extension seams below are the deliberately supported places to add policy or presentation.
Do not subclass an unlisted helper merely because its header is transitively public; use it as a
value/query type or compose it through a listed owner.

## Stable entry headers

| Domain | Primary headers | Intended use |
| --- | --- | --- |
| Authoring | `Authoring/RTSContentSet.h`, `Authoring/RTSContentSetScenario.h`, `Authoring/RTSGeneratedContentManifest.h` | Define vendor-neutral units, buildings, resources, teams, presentation profiles, and starter scenarios; adjust generated unit art through `FRTSUnitDefinition::VisualRotation` and `VisualOffset`; inspect exact generated-base provenance and ownership. Editor generation is exposed by the editor module, not runtime code. |
| Core identity | `Core/RTSOwnerComponent.h`, `Core/RTSGameplayTagsComponent.h`, `Core/RTSSelectableComponent.h`, `Core/RTSNameComponent.h`, `Core/RTSPortraitComponent.h`, `Libraries/RTSGameIdentityLibrary.h`, `Presentation/RTSTeamColorComponent.h` | Compose project-owned actors and consume their replicated identity/presentation state; resolve project branding and configure ownership markers or opt-in material tint for replacement art. |
| Player and match | `Player/RTSPlayerController.h`, `Player/RTSPlayerControlsSubsystem.h`, `Player/RTSSelectionComponent.h`, `Camera/RTSCameraControlComponent.h`, `Match/RTSGameMode.h`, `Match/RTSGameState.h`, `Match/RTSPlayerState.h`, `Match/RTSTeamInfo.h`, `Match/RTSGameSessionSubsystem.h` | Use the controller's high-level request, policy, targeting, and event facades; access its selection and camera services through `GetSelection()` and `GetCameraControl()`; implement authoritative objectives; and observe replicated match/team state. Use the session subsystem for the bundled menu and match travel lifecycle. The owning LocalPlayer controls subsystem supplies cached camera preferences, staged native keyboard bindings, verified Apply and change notifications for custom settings widgets. |
| Orders | `Orders/RTSOrder.h`, `Orders/RTSOrderData.h`, `Orders/RTSOrderTargetData.h`, `Orders/RTSOrderSubmissionComponent.h`, `Libraries/RTSOrderLibrary.h` | Add order classes, validate targets, and submit generic selected-unit requests through the controller-owned component returned by `GetOrderSubmission()`. |
| AI | `AI/RTSPlayerAIController.h`, `AI/RTSAIDifficulty.h`, `AI/RTSPawnAIController.h`, `AI/RTSAIBotSubsystem.h`, `AI/RTSSquadSystem.h`, `AI/RTSScoutSystem.h`, `AI/RTSCombatFrontageSubsystem.h` | Add or replace strategic policy; consume the bundled tactical/squad/scouting services and read the authority-owned attack-frontage assignment contract. |
| Combat | `Combat/RTSAttackComponent.h`, `Combat/RTSProjectileTargetComponent.h`, `Combat/RTSHealthComponent.h`, `Combat/RTSAbilityComponent.h`, `Combat/RTSResearchComponent.h`, `Combat/RTSStanceComponent.h`, `Combat/RTSUpgradeComponent.h` | Compose combat actors, configure vendor-neutral outgoing origins and incoming projectile target points, and override narrow validation/effect policies while retaining authoritative transactions. |
| Economy | `Economy/RTSPlayerResourcesComponent.h`, `Economy/RTSGathererComponent.h`, `Economy/RTSResourceSourceComponent.h`, `Economy/RTSResourceDrainComponent.h`, `Economy/RTSSupplyComponent.h` | Add resources, gathering policy, deposits, drains, and supply rules. |
| Construction and production | `Construction/RTSBuilderComponent.h`, `Construction/RTSConstructionSiteComponent.h`, `Construction/RTSFootprintComponent.h`, `Production/RTSProductionComponent.h`, `Production/RTSProductionCostComponent.h` | Add builders, sites, placement footprints, products, queue policy, payment, cancellation, and completion. |
| Containers | `Containers/RTSContainerComponent.h`, `Containers/RTSContainableComponent.h` | Add transports, bunkers, cargo policy, and custom unload placement. |
| Navigation | `Navigation/RTSFlowFieldSubsystem.h`, `Navigation/RTSFlowMovementComponent.h`, `Navigation/RTSFlowCostVolume.h`, `Navigation/RTSNavigationSystem.h`, `Navigation/RTSRecastNavMesh.h` | Request shared flow fields, add flow-driven units/cost regions, or select the paired RTS navigation-system/Recast integration. `ARTSRecastNavMesh` owns the `DynamicModifiersOnly` class default required by runtime building footprints in blank projects; C++ navigation-system subclasses can override `ResolveRTSNavigationDataClass` for a qualified replacement. |
| Vision | `Vision/RTSVisibleComponent.h`, `Vision/RTSVisionComponent.h`, `Vision/RTSVisionVolume.h`, `Vision/RTSFogOfWarActor.h` | Compose visibility sources/targets, configure height-aware vision, and consume local fog presentation. |
| Messaging | `Messaging/RTSChatComponent.h`, `Messaging/RTSPingComponent.h` | Bind chat/ping UI and extend server-side sender/recipient policy through the owning controller. |
| Skirmish | `Skirmish/RTSSkirmishDefinition.h`, `Skirmish/RTSSkirmishGameMode.h`, `Skirmish/RTSSkirmishSubsystem.h`, `Skirmish/RTSSkirmishLobbyState.h` | Define and launch data-driven skirmishes, and consume the authoritative replicated lobby roster and readiness state. |
| UI and feedback | `UI/RTSHUD.h`, `UI/RTSHudStyle.h`, `UI/RTSCommandCardLayout.h`, `UI/RTSCommandCardWidget.h`, `UI/RTSInfoPanelWidget.h`, `UI/RTSGameMenuWidget.h`, `UI/RTSGameViewportClient.h`, `UI/RTSSkirmishSetupWidget.h`, `Feedback/RTSEventFeedSubsystem.h` | Reskin the bundled HUD, replace panels, author command layouts, or bind replicated gameplay data to project widgets. Derive the native game viewport to extend its hardware/software cursor fallback while preserving customer cursor widgets. |
| Diagnostics | `Debug/RTSDiagnosticsSubsystem.h`, `Debug/RTSMatchCheckSubsystem.h`, `Debug/RTSPerformanceStats.h` | Add diagnostic channels/events, integrate packaged match qualification, and read or contribute to the documented capture-window performance counters. Counter publication requires the active capture scope and never authorizes gameplay. |

Supporting enums, structs, order subclasses, widgets, libraries, and data types beneath `Public/`
are supported where they appear in these entry-point signatures. Their header paths are stable for
the v1 contract after API freeze, but they are not an invitation to replace internal orchestration.

## Extension versus replacement

Use composition first. Add the relevant exported components to a project-owned Actor/Character, or
derive a customer-owned class from a generated project class. Store that subclass outside the
generated output root and select it through a customer-owned roster/map/configuration seam while
keeping its generated parent expected. A complete class supplied through a Content Set
`Existing*Class` field must instead derive from a stable native or project-owned class, because that
field suppresses the corresponding generated output. Never modify a manifest-owned base. Prefer a
narrow policy hook over replacing its transaction.

| Seam | Additive extension | Complete replacement |
| --- | --- | --- |
| Strategic AI | Leave `ShouldUseBuiltInStrategy()` true and add decisions in `ExecuteStrategyThink_Implementation()`. | Return false from `ShouldUseBuiltInStrategy_Implementation()` and implement the entire authority-side policy in `ExecuteStrategyThink_Implementation()`. The project then owns build, economy, research, ability, squad, and scout decisions. |
| Orders | Derive `URTSOrder`; override eligibility, target validation, description, and `IssueOrder_Implementation()`, calling the parent where it supplies required behavior. | A custom order class may own its execution, but it does not replace the controller's high-level facades, class-allow policy, targeting/events, the controller-owned submission component's admission and transport, or server-side target revalidation. |
| Ability | Override `ValidateAbilityReadiness`, `ValidateAbilityTarget`, or `ApplyAbilityEffect`; retain the component's cost/cooldown/replication transaction. | Replacing the complete ability component transaction makes the project responsible for authority, ownership, payment, cooldown, target security, replication, and notifications. |
| Combat/economy/production | Override documented `Can...` policy or presentation hooks and call the parent mutation. | Replacing `UseAttack`, health/death, payment, gathering, construction, production, research, stance, containment, or ownership mutations transfers every invariant in the feature guide to the project. |
| Match | Derive `ARTSGameMode`; call `EliminatePlayer` and `CommitMatchResult` from project objectives. | A replacement game mode must preserve replicated phase/result truth, roster lifecycle, reconnect/travel behavior, and exactly-once match completion. |
| UI | Derive a bundled widget/HUD for small changes, or bind documented component/subsystem delegates in a project widget. | Replacing the whole HUD is supported; gameplay remains in runtime components/controllers and the replacement must not inspect generated widget hierarchies as state. |
| Navigation/vision/networking | Configure and compose the exported systems. | Alternative navigation, fog, or replication architecture is outside the v1 qualified contract unless the project assumes the entire subsystem and requalifies scale, security, relevancy, and lifecycle behavior. |

## Native rules

- Implement `BlueprintNativeEvent` overrides through `_Implementation`.
- Call the parent for additive behavior. Skip it only for a complete-replacement seam named above
  or in the feature guide.
- Gameplay mutations are game-thread and authority-only. Route client intent through a supported
  high-level `ARTSPlayerController::Issue...` facade. For a generic selected-unit `FRTSOrderData`,
  use `GetOrderSubmission()->IssueOrderToSelectedActors()`. These request results report
  admission/submission, not eventual completion.
- Treat returned arrays/structs as snapshots and UObject references according to Unreal lifetime
  rules. Do not retain raw pointers across frames or teardown.
- Bind to public delegates/replicated state rather than polling private implementation details.
- Include only the domain header used by the translation unit. The module is maintained with full
  IWYU support and is qualified without unity, shared PCH, or transitive-include assumptions.

`CPP_EXTENSION_CONTRACT.md` is normative for threads, authority, lifetime, delegates, async work,
replication, and override safety. Feature guides define the exact transaction order and events.
`PUBLIC_API_CHANGE_POLICY.md` defines the v1 compatibility and baseline-update rules.
