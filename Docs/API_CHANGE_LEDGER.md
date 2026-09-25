# API change ledger

Every intentional public API change, newest first. Required by `PUBLIC_API_CHANGE_POLICY.md` before
a fingerprint is re-baselined.

An entry answers three questions: **what** moved, **why** it was worth moving, and **what a customer
would have had to do** about it. The third column is written even though no customers exist yet —
it is the honest measure of how disruptive a change is, and at `v1.0.0` those answers become
migration notes.

Fingerprints are recorded as they stand *after* the change.

---

## First-match experience and hosted HUD presentation

**Date:** 2026-09-24 · **Stage:** pre-release

New `AI/RTSAIDifficulty.h` adds `ERTSAIDifficulty` (Passive, Easy, Normal).
`ARTSPlayerAIController::GetDifficulty`/`SetDifficulty` read and set it. Passive never attacks; Easy
attacks later with bigger armies and its scout never harasses workers. `URTSAIBotSubsystem` gains
`EasyAttackArmyMultiplier` and `EasyEarliestAttackSeconds`. `URTSScoutSystem::TickScout` gains a
defaulted `bAllowWorkerHarass` parameter. `ARTSSkirmishGameMode::ConfigureAIDifficulty`,
`GetAIDifficulty` and `GetDefaultAIDifficulty` let setup choose it. `FRTSSkirmishDefinitionData` and
`FRTSStarterScenarioDefinition` add `DefaultAIDifficulty` (default Easy). The game mode's own default
stays Normal, so matches started without setup behave as before. Supported opponent counts include 0
(sandbox) when the map allows it.

`FRTSHealthProfile` and `URTSHealthComponent` add `bGathererFightsBack`, surfaced on the Content Set
unit as "Worker Fights Back" (default off). `ARTSPawnAIController::IssueDefensiveAttackFromWork` fights
the attacker and then resumes the interrupted gathering or return trip.

`URTSAttackComponent::ChaseRadius` is now enforced. Content Sets already wrote it and the docs already
described it, but nothing read it, so a unit that attacked on its own chased without limit: workers
that fought back followed a fleeing scout into the enemy base and died there. A unit that attacks on
its own (answering an attack, picking a target while idle, a worker fighting back, or an engagement
during attack-move or patrol) now gives up a target out of weapon reach once it is farther than its
chase radius from where the fight began. A worker goes back to its mining or return trip, an
attack-move or patrol continues its route, and an idle unit walks back to where it was. Attacks the
player orders are chased any distance, as before. Projects that relied on unlimited automatic chasing
can raise **Attack Chase Radius** in the Content Set or `ChaseRadius` on the attack component.

An idle worker (a pawn with a gatherer) no longer starts fights: `FindTargetInAcquisitionRadius`
returns false for it, so it waits for work and fights only when attacked (**Worker Fights Back**) or
when the player sends it on attack-move or patrol. The AI's base regions no longer overlap.
`FRTSSquadSystem::IsInBaseRegion` counts a position as one of our regions only when no other
player's known base is closer, and `FScanContext::ForeignBaseLocations` and
`FSeenEnemy::bResourceDrain` supply those bases. Before, on maps where bases sit closer than twice
the defense radius, each AI took its neighbor's own mineral line for an intrusion and sent its
workers there. In a four-player free-for-all, two AIs lost every worker within three minutes.

`URTSSelectionComponent` drops a selected actor the moment its health component reports it dead,
not when the body is removed. The built-in info panel shows the unit description under the context
heading for a single selection, puts a weapon's targets and range on their own line, and names the
attack cooldown bar in its tooltip instead of a caption. The controls hint sizes its key column to
the text, so a short combination never wraps. The description shows only when the context area has
no production, construction, research or node progress to show; the unit name carries it as a
tooltip. Empty progress lines, the idle progress bar and an empty rally line collapse, and the
product icon is 64 px, so the queue slots fit the console. Every Complete starter building now has
a description.

`ARTSPlayerController::SetObserver(true)` on a player still in the match now surrenders for them
first, so the match ends when one participant remains. Before, the player kept a place in the match
it could no longer act in, and a free-for-all never ended. Defeat still turns a losing human into an
observer, now through `ARTSPlayerState::SetObserver`. The melee elimination log line appears only
when the elimination happens.

AI worker defense drafts workers only into a fight the region can win: when the intruders' combined
strength exceeds that of the local military and draftable workers that can hit them, no workers are
drafted and drafted ones return to mining. A lone raider or a worker rush is still answered. The
skirmish game mode reads a new `?AIDifficulty=Passive|Easy|Normal` travel option for matches started
without the setup screen.

`ARTSGameMode` overrides `InitNewPlayer` and `ChangeName`. Humans are named "Player N" unless the
`RTSName` travel option names them; the online subsystem's nickname (often the host name) is no longer
shown. Requested names are trimmed, stripped of control characters and limited to 20 characters.
`URTSSkirmishSetupWidget` saves only a name the player typed; the name the server assigned is used
in the match but not saved for later sessions.

HUD: `URTSHudStyle` adds `UnitBarRelationshipFrame` and `SandboxObjective`, and lowers the defaults of
`FogUnknownOpacity` (0.92 to 0.55) and `FogKnownOpacity` (0.55 to 0.35); both are now applied to the
minimap fog. `URTSSimpleHealthBarWidget` adds `SetRelationshipFrameColor`, `GetHealthPercentage` and
`GetRelationshipFrameColor`. `URTSHealthBarWidgetComponent` pushes its first state once the health
component has begun play. `URTSSelectableComponent` colors the selection circle through the material's
new "Color" parameter. `ARTSHUD::bShowUnitNames` now defaults to false and labels use the HUD style's
font and relationship colors. `URTSCommandCardLayout` adds `FallbackInputActionNames`.
`GetInputActionForSlot` returns only positional slots, never a pinned command's key.
`URTSCommandButtonWidget` adds hover and focus notifications and a hosted tooltip.
`URTSCommandCardWidget::SetTooltipHost` shows tooltips above the card. `ARTSFogOfWarActor` adds
`ExploredBrightness`, `ExploredSaturation`, `UnexploredBrightness`, `EdgeSoftness` and
`ConfigureFogAppearance`. `URTSHudStyleSettings` becomes the "Real-Time Strategy" page under Plugins
and adds the editor-only `GameContentSet`.

Designer tooltips on `FRTSAttackData`, `FRTSAbility`, `FRTSEnergyProfile`, `FRTSStance` and `URTSAnimSet` are rewritten.

`URTSCheatManager` functions are `BlueprintProtected`: they remain console commands and callable in a
Blueprint subclass of the cheat manager, and no longer appear in other Blueprints.

Generated Blueprints name their components `<Thing>Component` (for example `HealthComponent`) instead
of `<Thing>C`; Generate renames the old components in place. Blueprints that referenced the old
variable names by name must use the new names.

Reviewed public-header fingerprint: `117328E10C8CA288BAC9C03B333CCFB0F9B330C7` (182 paths, every one now opening with the copyright notice).
Reflection: `06D9A0572E6A5C88F81B0EF0EFAADA31C2DB6706` (2,092 records).
Coverage: `RTS.AI.WorkerDefense.FightsBackAndResumesMining`, `RTS.AI.WorkerDefense.ChaseRadiusEndsAutomaticPursuit`, `RTS.AI.WorkerDefense.IdleWorkersDoNotStartFights`,
`RTS.AI.WorkerDefense.NeighborMineralLineIsNotAnIntrusion`, `RTS.UI.InfoPanelDescriptionAndDeadSelection`, `RTS.Framework.MatchState.ObservingForfeits`, `RTS.Skirmish.Factions.DifficultyTravelOption`,
`RTS.AI.WorkerDefense.NoDraftIntoHopelessFight`, `RTS.Skirmish.Factions.SandboxAndDifficulty`,
`RTS.Match.PlayerNames`, `RTS.Skirmish.Factions.SetupRemembersChosenName`,
`RTS.UI.RelationshipPresentation`, `RTS.UI.CommandCard.HotkeysShownInMatch` and
the migration and example-graph cases in `RTS.ContentSet.AuthoringExperience`.

---

## Content Set authoring for game developers

**Date:** 2026-09-24 · **Stage:** pre-release

Unit and building definitions gain `CustomBlueprintClass`: a customer-owned child of the generated
`BP_<Id>`. Generation keeps regenerating the parent and gives players the child everywhere a class is
spawned or offered (production, construction and starting bases); tech requirements keep naming the
parent, which children satisfy. `ExistingUnitActorClass` and `ExistingBuildingActorClass` keep their
meaning (a complete replacement the generator no longer updates) and are now labelled "Replace With
Class". Nothing to do for existing assets; the new field defaults to empty.

Content Set property categories move from `RTS|Authoring` to purpose groups (`RTS|Identity`,
`RTS|Appearance`, `RTS|Stats`, `RTS|Combat`, `RTS|Economy`, `RTS|Production`, `RTS|Construction`,
`RTS|Abilities`, `RTS|Blueprint` and, on the asset, `RTS|Game`, `RTS|Units`, `RTS|Buildings`,
`RTS|Resources`, `RTS|Match Setup`, `RTS|Gameplay Defaults`, `RTS|Presentation`, `RTS|Advanced`).
Reference fields gain display names ("Faction", "Required Buildings", "Trains Units") and pick from
the asset's own ids through new non-Blueprint `Get*IdOptions` functions on `URTSContentSet`. Tooltips
are rewritten for designers. `ResetToStarterRuleset` is no longer a details-panel button; the editor's
Reset asks first and restores the installed starter game instead. Enums with prefixed entry names
(`ERTSPaymentType`, `ERTSOrderType`, `ERTSVisionState` and six more) gain readable display names.
`FRTSSkirmishDefinitionData::GetBaseLocation` is the single base-ring layout used by the match and
the generated map's editor markers. Property names, types, defaults and serialized data are unchanged,
so Blueprint pins keep working; only their category, label and tooltip text changed.

Reviewed public-header fingerprint: `4F1322201B4AA0F0E957008EA165CF975BFFC7CE` (181 paths).
Reflection: `A6CC023154612A09672006B2F6DBC45165D8B4E4` (2,071 records).
Coverage: `RTS.ContentSet.AuthoringExperience` generates a starter set, assigns Custom Blueprints,
regenerates, and checks spawn classes, inherited values, validation, id options, text localization,
the generated map's editor camera and markers, and volume brushes.

---

## Gameplay timing and standalone analytics ownership

**Date:** 2026-09-12 · **Stage:** pre-release

Production and research retain precise private work counters and the last observed simulation time.
Native completion requires all authored work; a small positive remainder no longer grants early
completion. Work excludes time preceding admission and unpaid production intervals while preserving
pause, actor dilation, production advantages and completed-but-blocked retries. Published float progress
and reflected queue fields retain their types; intentional state edits resynchronize private work.
Only private state and helpers change in the production and research public headers.

Explicit analytics CSV output is acquired by the first gameplay-world snapshot. Unreal's temporary
standalone startup world no longer reserves the file before the playable map loads. A subsystem
appends only after creating its own output, and a replacement world rejects an already committed
explicit path instead of extending another world's capture. Existing generated per-world names
remain unchanged. Two private, non-reflected ownership/rejection flags change the diagnostics layout.
Use a fresh explicit path for a new capture; arbitrary travel does not continue a previous capture.

These private layout changes require native consumers to rebuild with the plugin. No callable
signatures, reflected properties, header paths, Blueprint assets or serialized schemas change.
Reviewed public-header fingerprint: `A91B1B8C5B00F576A277C75A8125E9BC40A573AF` (181 unchanged paths).
Reflection remains `FA1FA5F54BE165D6F3BD335092BB5A7702ED4353` (2,069 records).
Regression coverage uses actual standalone initialization, world-tick output and native paid
production/research. Strict Mac compilation and all 164 tests in a fresh installed copy pass with
zero warnings, including those regressions and networked lobby keyboard interaction. Packaged-game
and release qualification remain separate gates.

---

## Research labels and optional cursor fallback viewport

**Date:** 2026-09-12 · **Stage:** pre-release

Existing authoring and runtime research options gain optional localized `DisplayName` and
`Description` fields. Generator 12 copies them into owned output; names and descriptions now feed
command cards, cancellation, prerequisites and owner-bound completion messages. Stable upgrade
keys, costs, effect categories and network events remain unchanged. Empty or conflicting names
fall back to the stable key. Preview and regenerate project-owned Content Sets to adopt the new
generated metadata; no source-schema migration is required. Custom existing research classes can
set the optional runtime fields directly.

New `UI/RTSGameViewportClient.h` supplies a native viewport that uses hardware fallback for cursor
types without a software widget and preserves supplied widgets and the software-enable policy.
Starter installation configures it only for stock viewport defaults, preserving configured,
cached and live customer classes. Installation explains when an editor restart is required; it
does not replace a running viewport. Existing custom viewports can derive from this native class
or retain their current policy. The new header is covered by a separate external-consumer unit.

Source and independent patch review precede baseline acceptance. Reviewed public-header
fingerprint: `4B3ACD5880445287ECEDB41CED99029BAE210D2F` (181 paths; 76 advertised entry headers).
Actual compiled73 reflection is `FA1FA5F54BE165D6F3BD335092BB5A7702ED4353` (2,069 records):
exactly one viewport class and four optional text fields added, with no removed or altered prior
records. The native cursor test passes without warnings; paid research presentation passes with
eight unchanged fixture setup warnings. Generator12 completed twice in an isolated host and passed
a separate cold native-component check. Exactly seven of278 assets changed; all271 unrelated
assets were preserved. Rendered startup qualification remains pending. Private research access and the test-only command-card friend
do not add callable customer APIs.

---

## Pointer targeting ignores authored non-target volumes

**Date:** 2026-09-12 · **Stage:** pre-release

The actual remapped Move regression exposed a command accepted at the top of an invisible camera
bounds volume, leaving the worker stationary. Pointer-confirmed commands now share the existing
`DefaultOrderIgnoreTargetClasses` policy with default right-click orders, before both actor-target
and location-fallback passes. Actor-first rally behavior is retained. The existing property keeps
its name and type; its comment now documents both consumers. Customers can continue configuring
the same ignore list, which now also applies to armed pointer commands.

Reviewed public-header fingerprint: `67A31112B02C2A30B19D74A1A7E335012747FE54` (180 unchanged paths).
Reflection remains `1278E83F2E4EC0D6D328F01521F6DB67F926C79E` (2,064 records).
The native regression compares the accepted destination with an independent terrain hit through
the actual volume, then requires real worker movement. Compiled keyboard72 acceptance passes without errors or warnings.

---

## Native keyboard rebinding and owning-player hints

**Date:** 2026-09-12 · **Stage:** pre-release

The existing player controls subsystem adds `FRTSKeyboardBinding`, `FRTSKeyboardControls`,
`GetKeyboardControls`, `ApplyControlPreferences`, and `OnKeyboardBindingsChanged`. A captured
snapshot identifies the owning player's current input state; callers stage individual keys and
submit the complete snapshot. The combined controls operation validates the batch, preserves
unrelated native input mappings, and verifies saved preferences before reporting success.
Existing camera-only methods remain available. `InputCore` becomes a public module dependency
because the exported snapshots contain `FKey`; `EnhancedInput` remains an implementation dependency.

The bundled mapping context gains stable native player-mappable identities on nine existing
keyboard actions. All 86 authored action/key/trigger/modifier relationships are preserved. Customer
contexts are read through their actual owning controller and are not modified at runtime. Custom
layouts can retain the older camera-only controls or add the optional keyboard panel. Command hints
resolve the owning player's effective bindings, including the authored selection/queue action.

Rebuild matching binaries and refresh the bundled mapping asset when updating the plugin. Projects
with replacement contexts opt into native player-mappable rows through their own asset metadata.
Project-owned persistence policies remain intact; the stock menu explains when it cannot safely
persist those bindings. Source review retains all 180 public paths. The reviewed public-header
fingerprint is `A2AB4E951A57708E68DB88B0C1A67DE20E500D4D`. The controller adds one private friend for
assigned actions; UI header changes add private binding state and callbacks. Actual compiled70
reflection is `1278E83F2E4EC0D6D328F01521F6DB67F926C79E` (2,064 records): exactly two structs,
nine struct fields, two Blueprint functions and one assignable delegate property were added, with
no removals. Strict70 passes all 653 source files unchanged. Gameplay acceptance remains in
progress; the initial native test exposed a movement-readiness timing check requiring investigation.

---

## Persistent local camera preferences

**Date:** 2026-09-12 · **Stage:** pre-release

The existing Settings panel adds staged edge scrolling and scroll speed controls. Applying verifies
both preferences on disk before changing the cached local controls; Cancel leaves them unchanged
and Restore stages camera defaults. The multiplier preserves authored camera speed, zoom scaling,
keyboard pan and world bounds. These preferences are shared by local players in the installation
and do not affect replicated match state.

New `Player/RTSPlayerControlsSubsystem.h` exposes the owning LocalPlayer's cached getters,
`ApplyCameraPreferences` with an actionable error, and `OnCameraPreferencesChanged`. Existing custom
menus can bind these APIs; their older widget trees remain supported without the optional camera
controls. `UI/RTSGameMenuWidget.h` gains private helpers/callbacks only. No assets require migration;
rebuild matching binaries and include the new domain header when using the subsystem.

Source review preceded baseline acceptance. Reviewed public-header fingerprint:
`D44260DF16665C33C1EFCFBD4D4A414BA24BBD4C` (**180 headers**). The existing native edge test
moves with its private camera-input implementation, retaining ue4-rts lineage and the same158-test
inventory. Actual66 compiled reflection is `83BB29B6D0AF5DA0169DF2C865F78A748A8BE2DC`
(**2,050 records**): exactly one class, three Blueprint functions and one assignable delegate property
are added. Review precedes baseline acceptance. Strict66 passes633 source files unchanged. Native
settings qualification exposed an isolated-file fixture being redirected by engine initialization;
that correction and rendered/new-process persistence remain pending. This entry does not establish
key rebinding.

---

## Construction readiness and automatic stationary combat

**Date:** 2026-09-12 · **Stage:** pre-release

Normally funded matches exposed unfinished towers firing and automatic retaliation selecting aircraft
outside a stationary weapon's reach. Native attack execution now requires construction readiness,
including revalidation after the customer admission callback; the stock Attack order blocks unfinished
sources. Weapon-domain capability queries retain their existing meaning.

Automatic acquisition and retaliation use stock order admission and actual weapon interaction reach
when the pawn cannot pursue. An owned automatic target is released after leaving reach; accepted
explicit replacement orders clear automatic ownership before notifying listeners. Mobile pursuit,
Hold and interrupted AttackMove/Patrol behavior retain their existing intent.

`AI/RTSPawnAIController.h` adds private automatic-target state, native helpers and a health-component
friend. Public signatures, reflected declarations, header paths and serialized assets do not change.
Existing acquisition and attack-admission code moves into cohesive private source files, retaining
its reviewed ue4-rts lineage. Rebuild matching binaries; customer assets need no migration. Customers
who allowed unfinished buildings to attack must now wait for native construction completion.

Reviewed public-header fingerprint: `EE2159F2F157E83ED8FFAA9BAF3F76561D870EE5`
(**179 headers**). Expected reflection remains `BBF8CCBEA90C64F595743CD526E80F9114F69D68`
(**2,045 records**), with the same **158-test** inventory.

Source review preceded fingerprint acceptance. Strict61 passes627 source files; compiled API
contracts match. Full61 exposes two fixture gaps (strategic reassignment of the controlled aircraft
and missing teams/vision), corrected in the actual passing63 combat regressions. Revision64 adds
real camera bounds to the generated-tower fixture. Strict64 and the exact installed158-test suite
pass with both combat regressions at zero errors/warnings. Two ordinary matches finish without
alarms or stalled orders; one tower completes but never engages, so that run adds no new combat proof.

---

## Stationary defenses and mobile squad eligibility

**Date:** 2026-09-12 · **Stage:** pre-release

Normally purchased towers exposed a squad integration defect: every ready armed pawn counted as
mobile infantry and could receive impossible march/chase orders. Squad admission now requires
actual locomotion: a working pawn movement component, its movable updated component and a finite
positive native maximum speed. Stationary weapons retain autonomous attacks and in-range local
worker-defense coverage. Deployed artillery remains eligible for normal stance handling. Stale
memberships are pruned through a per-scan set without changing support cooldowns or production caps.

The only public-header changes are a private stationary-combat scan field and a development-test
friend in `AI/RTSSquadSystem.h`. No public functions, reflected records, asset schemas or header paths
change. Rebuild matching binaries; customer data requires no migration. Custom moving structures
qualify through their actual movement component rather than class names or construction tags.

Reviewed public-header fingerprint: `75F1D48E33E3232A1513B73576F272296621C831`
(**179 headers**). Expected reflection remains **2,045 records**,
`BBF8CCBEA90C64F595743CD526E80F9114F69D68`. The **158-test** inventory remains
`f8f96fb864b04b92f2179289e83f7a31a89aa1114a1cd26482ab3f05b9fcc060`.
The existing paid-defense regression adds real squad admission and mobility-transition checks.
Review preceded baseline acceptance. Strict60, the exact installed158-test suite and the expanded
defense regression pass; compiled header/reflection contracts match. Two fresh normal matches confirm
mobile exclusion and actual Tower kills, then expose the combat issues addressed by the newer entry.

---

## Construction handoff and route-marker composition

**Date:** 2026-09-12 · **Stage:** pre-release

Native construction can accept a Continue order before the worker behavior tree assigns its site.
The AI progress watchdog now resolves that accepted same-world, same-player construction target
through the handoff. It retains the existing progress timeout; terminal targets are cleaned up
without falsely reporting a timeout. The real defense regression verifies that the original creating
worker reaches site assignment before its deliberate death and normal replacement recovery.

Numbered route markers now paint through the existing console widget's Slate layer, above
screen-space health widgets. Route lines retain the existing Canvas path. The console paints dynamic
markers independently of cached widget properties and reads only the existing sanitized accepted
plan, capped at twelve destinations. This replaces the insufficient Canvas-only depth adjustment.

The only public-header addition is a protected native `URTSConsoleWidget::NativePaint` override.
There are no new reflected declarations, header paths, assets or data schemas. Rebuild matching
binaries. Existing custom native console paint overrides should call `Super::NativePaint` to retain
the standard marker presentation; custom Blueprint console trees inherit it.

Reviewed public-header fingerprint: `DEFAC2BB8050281CC5C7F03A9DA4ED0E72EAF82F`
(**179 headers**). Expected reflection remains **2,045 records**,
`BBF8CCBEA90C64F595743CD526E80F9114F69D68`. The **158-test** inventory remains
`f8f96fb864b04b92f2179289e83f7a31a89aa1114a1cd26482ab3f05b9fcc060`.
Review preceded baseline acceptance. Strict59 and the exact installed158-test suite pass; actual
compiled header/reflection checks match these contracts. The focused defense test passes without
warnings. Rendered marker overlap and normal-match defense observation remain required.

---

## Native AI home defense and player-facing command details

**Date:** 2026-09-12 · **Stage:** pre-release

The built-in AI now discovers constructible defensive buildings by their working weapons and plans
one home defense after the economy has enough workers, a ready resource drain and a ready military
producer. Existing blocking construction goals keep essential technology and recovery ahead of this
optional purchase. Placement uses bounded native collision, worker-path, corridor and threat checks
and must cover the economic anchor. An infeasible optional goal releases its savings reservation;
admitted unpaid construction retains its existing reservation. Retry and retirement preserve the
exact tracked queue item and unrelated customer intentions.

The only public-header change adds private AI state/helpers and a development-test friend in
`AI/RTSAIBotSubsystem.h`. Existing public functions, reflected fields and extension signatures are
unchanged. Rebuild matching plugin binaries; no customer data migration, Content Set schema change,
generator change or asset regeneration is required. The original expansion-deferral function moves
unchanged between existing AI implementation files with its recorded lineage retained. New defensive
policy helpers are original implementation.

Reviewed public-header fingerprint: `1AE7D2FAF2895B41DD0FE6C8AA99F06AF11040E2`
(**179 headers**). No reflected declaration changes are proposed; the expected reflected contract
remains the actual55 value, `BBF8CCBEA90C64F595743CD526E80F9114F69D68` (**2,045 records**).
This review precedes baseline acceptance and does not replace compiled verification.

The registry retains all157 prior tests and adds only
`RTS.AI.StaticDefense.AcquisitionAndRecovery`: **158 tests**, inventory SHA-256
`f8f96fb864b04b92f2179289e83f7a31a89aa1114a1cd26482ab3f05b9fcc060`.
It exercises real navigation, independently named authored classes, payments, construction work,
worker death/reassignment, caps and hostile-aircraft combat. Execution remains required. The existing
network queue fixture also restores persisted editor play preferences after its real network run.

Private presentation changes expose authored unit/building descriptions in existing purchase
tooltips and draw bounded waypoint labels above world annotations while retaining route-line depth.
They add no API or assets. Actual higher-resolution rendering must verify the overlap correction.

---

## Accepted command plans and queue-capacity feedback

**Date:** 2026-09-12 · **Stage:** pre-release

A player inspecting one owned live pawn can now see the server-accepted current command and pending
queue, with numbered visible destinations and capacity feedback. The owner-only snapshot carries
no target actor reference; unseen target locations are omitted. Inspection is bounded, retries
budget-rejected requests, and clears across selection, ownership, observer and death transitions.
Custom Blueprint HUDs can bind `OnAcceptedOrderPlanChanged` and read `GetAcceptedOrderPlan`.

This adds `FRTSOrderPlanStep` and `FRTSAcceptedOrderPlan`, their read-only fields, the snapshot getter
and assignable event, private inspection/replication callbacks, and the native Canvas entry point.
The private batch-completion RPC gains capacity-rejection fields. Existing public order submission
and `EnqueueOrder` bool APIs retain their signatures and semantics; queue capacity is unchanged.
Rebuild and package matching client/server binaries together. No Content Set schema, generator,
asset regeneration or customer-authored data migration is required.

The reviewed public C++ header fingerprint is `399771A184EB08E6385C788134061C5F5A980C98`
(**179 headers**). Five public headers changed: the new snapshot header, submission component,
pawn controller (private helpers), world-cue subsystem and native info panel. The actual compiled49 reflected
capture adds exactly 16 Blueprint records: two structs, their twelve read-only fields, the getter
and the assignable event. It removes or changes no existing Blueprint record. The reviewed reflected
fingerprint is `BBF8CCBEA90C64F595743CD526E80F9114F69D68` (**2,045 records**). Private callbacks/RPC
plumbing remains covered by header review. This review precedes reflected-baseline acceptance;
native gameplay and rendered acceptance remain required.

The first full installed run caught eleven missing field tooltips. The reviewed follow-up adds
customer explanations without changing declarations, property flags or the reflected fingerprint;
the header fingerprint above includes these comments. Native network-test replica lookup now uses
actual NetGUID identity instead of dynamic actor names; its deadlines and assertions remain intact.

The registry retains all 156 previous tests and adds `RTS.Orders.AcceptedPlanNetworkPrivacy`;
**157 tests**, inventory SHA-256 `e32be72bbbd2502543ae9a906b0c25d45c4a40f1b3ec43bdf1c0ec99801a8967`.
The existing waypoint test also gains actual accepted-plan advancement assertions. The new test
uses real PIE network worlds and owning RPCs, including saturated-budget retry, hidden destinations,
capacity, ownership and native HUD lifecycle. Compilation and execution remain required.

---

## Resource placeholder silhouettes — generator 11

**Date:** 2026-09-12 · **Stage:** pre-release

Automatically generated sources now have distinct lightweight ore clusters and gas vents. Their
authored collision and navigation dimensions remain unchanged. The original collision body stays
hidden; visible decoration does not collide or affect navigation. Customer-assigned meshes and
existing source classes retain their own presentation. Regeneration removes only tagged generator
decoration and preserves customer attachments, rejecting unsupported attachment chains explicitly.

Generator version advances **10 → 11**; Content Set schema remains **2**. Customers review the
upgrade notice and regenerate owned resource-source outputs, then cook their game again. No new
asset package, reflected type, or serialized authoring field is added. Rebuild native consumers
for the updated generator-version constant.

Reviewed header fingerprint → `3169FB94D2783A16D99B6A1ECF41F2EAA1088034` (**178 headers**).
The only public-header edit is the generator constant. Reflection remains the actual Build44
contract: **2,029 records**, `021802E0484DC1E28E05C34A5140262A86CE8533`. Existing navigation and
convergence tests add real decoration, bounds, fog hide/show and customer-attachment checks; the
exact **156-test** inventory remains unchanged. This review precedes rebaselining. Strict compilation,
regenerated assets, native execution and rendered acceptance remain required.

The same sprint fixes commandlet cleanup ownership: saved generation already retires its detached
world. Both commandlet cleanup paths now retire only a still-initialized detached world, preserving
editor-owned and already-cleaned assets. This eliminates duplicate teardown without suppressing
warnings or changing public APIs.

---

## Team vision after map startup — compatible runtime correction

**Date:** 2026-09-11 · **Stage:** pre-release

`EnsureTeamAndVisionForPlayer` already supports team counts beyond the initial `NumTeams` default.
Its new team vision actor previously remained outside a running vision manager, leaving those teams
without an initialized grid. This affects larger setup selections made after map startup and native
extensions that admit additional teams.

The correction admits a compatible grid into the existing manager, initializes only a new grid,
and refreshes stationary observers when team membership changes. It preserves explored terrain and
the local player's fog display. Private registration helpers and friendship declarations change
header text without adding a reflected function, property, or serialized class. No generated-content
schema/version change is required. Existing callers retain the same API; rebuild native consumers.
Fog-less maps retain valid team membership with explicitly uninitialized pending grids. When their
volume becomes usable, the manager initializes those grids without clearing existing exploration.

Reviewed header fingerprint → `2F9E5A5915719B2007CB90947F6E7C97ED50FE5D` (**178 headers**).
The two changes are private declarations in `RTSVisionInfo.h` and `RTSVisionManager.h`; the
compiled Build40 reflected contract remains **2,029 records**,
`021802E0484DC1E28E05C34A5140262A86CE8533`, with no reflected additions or removals expected.
The source registry retains all 154 previous tests and adds real runtime team admission and Hold
Position combat coverage (**156 tests**, SHA-256
`3f08c4d56cb98362bacc724e9e9c83a546d46a99cd5d24cb1a41cd55fd5e2482`).
This source review precedes rebaselining; strict compilation and installed automation remain required.

The same sprint corrects native Hold Position acquisition: the idle behavior tree must not publish
a pursuit target while Hold is active. Existing range-only attacks remain enabled, and a replacement
Move still releases Hold. This is a private implementation correction with no additional header,
reflection, or generated-content change.

---

## Generated flight and network resources — generator 10

**Date:** 2026-09-11 · **Stage:** pre-release

Generated air units now apply `MoveSpeed` to `CharacterMovement.MaxFlySpeed` as well as walking
speed. Previously an authored Bomber speed of 330 and Interceptor speed of 550 both flew at the
engine default of 600. Zero restores the respective engine defaults; removing the air capability
also restores the engine flying default. The generator owns these movement properties on its
outputs. Existing customer-supplied unit classes remain outside generation, and project-owned
extensions retain their normal ownership protection.

Generated neutral resource actors now enable actor replication. Their replicated resource components
previously sat on non-replicated `AActor` defaults, so authority-created minerals and geysers never
reached remote players. Existing per-player visibility routing remains authoritative; sources do not
become always-relevant or owner-only. Existing customer-supplied source classes remain untouched.

Generator version advances **9 → 10**; Content Set schema remains **2**. Review the generator-upgrade
notice and regenerate owned unit and resource-source outputs, then rebuild/cook. No schema migration is required.
The `MoveSpeed` authoring tooltip now describes both movement modes; no field is renamed or removed.

Native flight orders use swept capsule routes and an internal precise path follower. The controller
adds private flight state, while the AirUnit header corrects its separation documentation. Customer
path followers retain their object and authored settings; more-derived implementations are not
silently replaced. Scout and squad headers add private automation friends and clarify observation
memory. No public method/property or header path is removed or added. Rebuild native consumers for
the private controller layout change; existing serialized classes need no redirects.

Reviewed header fingerprint → `47AAC9733258C000204DAD0D54B8D801E6E87F47` (**178 headers**).
The canonical reflected inventory adds one internal class, `RTSFlightPathFollowingComponent`, with
no exposed members: expected **2,029 records**, `021802E0484DC1E28E05C34A5140262A86CE8533`.
The prior 2,028-record canonical baseline was reconstructed from the saved actual capture plus its
five reviewed additions and independently matches Build37's exact fingerprint. This entry precedes
rebaselining; UHT and the installed native API test must confirm the new expectation. Source
registrations retain all 150 prior tests and add exploration arrival, visible building memory,
scout observation memory, and real flight movement (154 total).

---

## Generated battlefield and match feedback — generator 9, additive API

**Date:** 2026-09-11 · **Stage:** pre-release

Starter scenarios append `MaximumParticipants` (default 16) and `ArenaPreset` (default OpenArena).
The new `ERTSStarterArenaPreset` enum also offers TwinCrossings, a two-player battlefield with a
central blocked island and two ground approaches. Generated skirmish definitions append the same
participant cap. Local setup, URL starts and authoritative lobbies reject oversized lineups before
spawning; connected spectators retain their identities. Existing definitions keep the prior cap and
flat arena. The editor-only native `ARTSMinimapVolume.SetEditorMinimapImage` assigns map-owned terrain
imagery through the existing field; no runtime actor observations are baked into that image.

Generator version advances **8 → 9**; Content Set schema remains **2**. Review the generator-upgrade
notice, regenerate owned outputs, and rebuild/cook maps; schema-2 assets need no schema migration.
Choose TwinCrossings explicitly with two players/cap two and sufficient map extent. New bundled
Complete authoring selects that preset. Customer-owned replacement assets retain existing ownership
protection. Native consumers must rebuild for the appended struct fields and private class changes.

Remote depletion feedback uses a private owner-only reliable RPC with authoritative per-player sight
and existing match identity/revision/start-time guards. It does not change the public gameplay
depletion delegate's networking semantics. AI composition adds private last-observed capability
state; reinforcement arrival retains existing squad priorities and the join distance. Private
helpers/friends and comments are reviewed header changes, with no removed public callable.

Reviewed header fingerprint → `CE5BA8AF0B90E3F967ED52E393AEC54C3D5C1027` (**178 headers**).
Expected reflection adds exactly three editable struct fields and one enum to the compiled Build34
baseline: **2,028 records**, `72FC26C0986E595FB4DCB40439E149299E49FB02`. Private RPCs remain excluded
from the public-callable fingerprint. This expectation is recorded before rebaselining; compiled
confirmation and actual regenerated/cooked-map verification remain pending.

---

## First research feedback and neutral sight — pre-release

**Date:** 2026-09-11 · **Stage:** pre-release

Native player controllers create their replicated upgrade ledger during construction, so the first
paid research completion has its feedback listener before incrementing the level. The local feed
retains weak references to the exact wallet and ledger it bound, and detaches those delegates when
components are replaced. Neutral visible actors no longer require an ownership component to enter
the existing sight and replication system. No public callable or serialized field changes.

The reviewed header delta contains two forward declarations and two private weak fields only.
Header fingerprint → `D3FB030CF9FB7D45224D29E642EB61010A0CB139` (**178 headers**).
Installed Build34 confirms the unchanged compiled reflection contract: **2,024 records**,
`C24C78A7F9389E7843F84FD2CA80129FEEFC448D`. Rebuild native consumers for the private layout change;
no content migration is required. The complete installed suite passes 146/146 tests.

---

## Contextual labels and readable minimap — additive presentation API

**Date:** 2026-09-11 · **Stage:** pre-release

`ARTSHUD.bShowAllUnitNames` is a new optional serialized default, false. With existing
`bShowUnitNames` enabled, the HUD labels the hovered actor and a single selected actor. Enable
`bShowAllUnitNames` to retain the previous always-labelled presentation; disabling `bShowUnitNames`
still hides every name. Labels use the owning local player's visibility/relationships and clear
collision bounds. The normal path inspects at most two actors rather than scanning the world.

The native minimap admits visible resource nodes, distinguishes bases/buildings/mobile forces,
preserves authored texture/material marker dimensions and frozen building observations, and paints
fog before admitted markers. `URTSVisibleComponent.GetClientVisionState()` is a native read-only
accessor for observed sight without the friendly-ownership exemption. The default empty map
background is brighter; authored background images remain unchanged. No public paths were removed.

Reviewed header fingerprint → `5FF837DF66B7C970A598CD97CFF301144132E4FF` (**178 headers**).
The expected reflection adds exactly the new HUD bool with the same EditDefaultsOnly/category flags
as `bShowUnitNames`: **2,024 records**, `C24C78A7F9389E7843F84FD2CA80129FEEFC448D`. This is a
reviewed expected additive contract, derived from the unchanged compiled baseline plus that one
record. The installed Build32 API test confirms this exact compiled fingerprint and header contract.
Rebuild native consumers for the HUD layout change.
Existing content schema/generator identities remain unchanged.

---

## Local audio, HUD layout and construction recovery — pre-release

**Date:** 2026-09-11 · **Stage:** pre-release

Local-player feedback now owns audio lifecycle across errors, HUD events, command confirmation and
match outcomes. It resolves that player's HUD style, honors explicit null order sounds, separates
sound from marker visibility and unbinds old/foreign-world callbacks. Five private reflected handlers
support existing gameplay delegates; no existing public callable is removed or changed. Bundled
original SoundWave/SoundCue assets use the existing authored sound fields. Buyer-owned nullable
settings remain unchanged; regenerate owned outputs to apply an intentional selection-sound edit.

`FRTSCommandCardEntry` appends the native `FallbackGlyph` text field for recognizable command paging.
The native info panel retains a private stats-host pointer for idle/selection layout. Their class and
struct layouts change, so native consumers must rebuild; no serialized-field migration is needed.
Construction now preserves scout reservations and recovers abandoned paid manual sites. Its only
public-header change is an automation friend. MatchCheck retains casualty evidence while deferring
WorkerSlaughter during its existing demonstrated occupied-collapse classification; private state and
helpers support reevaluation, and verdicts add `workerDefenseWatch` evidence per player.

Reviewed header fingerprint → `CE78E983B001F6AFDFC885EB975281E3F2825B04` (**178 headers**).
Compiled Build30 reflection is unchanged: **2,023 records**,
`796D8DBAD19B54D1A1DCF5AC9B89934063AD2141`. The five private delegate handlers are not public
callables and are excluded by the reviewed canonical API filter. The installed API test passed.
Existing Content Set schema2/generator8 identities and public header paths remain unchanged.

---

## Worker recovery and bounded base defense — additive native API

**Date:** 2026-09-11 · **Stage:** pre-release

`FRTSSquadSystem::TickSquads` retains its existing overload and adds an overload accepting a set of
workers reserved by the caller. Native bot scouting uses this to preserve active scouts during base
defense. Draft memory now privately retains the exact defense target and economic base so released
workers stop obsolete pursuits without replacing intervening player/custom orders. Worker recovery
uses actual trainable-worker costs and source capacity; its automation friend is the only bot-header
change. Existing callers need no source migration; rebuild native consumers for the private layout.

Reviewed header delta: one overload, private draft/base/reservation storage, and one test friend.
Header fingerprint → `AD7555819382BF35DC191DF34DCE576531090B4A` (**178 headers**).
Reflection is unchanged from the compiled branding baseline below (**2,023 records**).

---

## Project branding and HUD-safe opening — additive

**Date:** 2026-09-11 · **Stage:** pre-release

`URTSGameIdentityLibrary::GetGameDisplayName` exposes the existing Unreal project display name,
falling back to the internal project name. Native setup and game menus display this same identity.
The editor's `URTSContentSetEditorLibrary::SetGameDisplayName` and **Rename Game** action validate
and save the display name and literal window title together, preserving other project settings.
Content Set names remain ruleset metadata; executable, package and store identities remain Unreal
project settings. Existing customers need no data migration or additional independent branding field.

Native opening height changes from 4200 to 4600 (the existing zoom ceiling), with actual opening
geometry framed above the HUD on both opposing bases. Authored camera subclasses keep their settings.
Team-color timer ownership is now retained weakly so delayed component destruction cannot access a
world that has already gone away. These lifetime fields and camera helpers are private.

The reviewed combined header delta also includes generator 8's source-navigation change below.
Header fingerprint → `D106484A162B06483009917904189082602E6A6C` (**178 headers**).
Compiled reflection adds exactly the identity-library class and its Blueprint-pure getter, with
no changes or removals: `796D8DBAD19B54D1A1DCF5AC9B89934063AD2141` (**2,023 records**).
No existing runtime callable declarations were removed or changed; native consumers must rebuild.

---

## Finite resource-source navigation — generator 8

**Date:** 2026-09-11 · **Stage:** pre-release

`FRTSContentSetCustomVersion::CurrentGeneratorVersion` advances from 7 to 8; authored schema remains 2.
Generated mineral and gas source actors now have an unscaled scene root and an authored-size dynamic
null-area navigation box. Their physical mesh keeps collision but no longer exports static navigation.
This makes runtime-spawned sources block paths in modifier-only navigation and allows native depletion
or extractor absorption to reopen the same ground. The box adds no physical collider. Source visual
half-height offsets now operate relative to the scene root, independent of mesh scale.

Preview and apply the Content Set upgrade, then regenerate owned outputs and rebuild/cook affected maps.
This also removes old source geometry from baked navigation. ExistingSourceActorClass overrides remain
customer-owned and unchanged; their authors must provide removable navigation obstacles compatible with
the map's runtime generation mode. No runtime callable signature or reflected property changes.
The sole public-header change is the generator constant and source-size documentation; rebuild native
consumers. The reviewed fingerprint is recorded with the next coordinated compiled API check.

---

## Server-managed matches and replaceable starter presentation — additive

**Date:** 2026-09-11 · **Stage:** pre-release

Dedicated lobbies expose their server-managed status and accept an owning participant's
`RequestSkirmishLobbyReturn` only after a completed match. Readiness, lineup revisions, and
participant authorization remain authoritative. Customer UI can use the same rematch transaction.

`URTSTeamColorComponent` adds opt-in mesh-component/material-slot/parameter bindings for local
material instances, plus an explicit refresh operation. Empty bindings preserve custom art.
The minimap gains a Relationships/TeamPalette mode with its legacy mode as the default; the HUD
style gains an editable match objective. A native team-assignment event updates presentation,
and fog snapshots retain the observed team identity. No existing callable API is removed.

The native starter camera defaults to height 4200 and frames actual opening resources; authored
camera subclasses retain their defaults. AI technology waits for a configurable worker economy
floor. These tuning changes require no asset migration. Generator version 7 adds team bindings
only to generated placeholder parts and uses a separate starter floor material; assigned customer
meshes remain authoritative. Regenerate bundled/customer generated outputs to adopt these defaults.
Three targeting getter implementations moved out of line without signature changes. Other header
changes are private lifecycle, diagnostic, or automation state. Rebuild native consumers.

Reviewed header fingerprint → `027E3CBD3D42992BD270C25465FFFEF36B5F3C15` (**177 headers**).
The compiled reflected delta was reviewed: **12 additions, no changes or removals**.
Reflected fingerprint → `E0D1E6F245D4D819C3FB45EB94E43454B1D18061` (**2,021 records**).

---

## Gather navigation and late lobby identity — internal state

**Date:** 2026-09-11 · **Stage:** pre-release

Private controller methods and flags track a complete, in-range Gather approach and release its
behavior-tree pause on interruption. A private setup-widget flag detects identity arrival after an
unresolved lobby render. An automation-only friendship permits testing actual AI construction
reservations. No callable signatures, reflected properties, serialized values, or assets change;
customers need no migration. Rebuild native consumers with the updated plugin.

The reviewed header delta contains only these private declarations. Header fingerprint →
`2D62A7D1D65BE62D7477DC7CDF0887FBDAEC558A` (**176 headers**). The expected reflected contract remains
`E30A4723F3C621E177E60F157FA3E7EC4D7B77C8` (**2,009 records**), checked by the compiled API audit.

---

## Playable session flow, replicated lobby, and valid extractor placement

**Date:** 2026-09-11 · **Stage:** pre-release

The Blueprintable game menu exposes native settings, surrender, connection, and return-to-setup
flows through HUD/controller entry points. The BlueprintType Game Instance session subsystem keeps
connection status and errors across travel. The replicated lobby snapshot exposes factions, stable
participant bases, spectators, readiness, and lineup revisions; owning-controller requests validate
against the displayed revision. `IsActivePlayer` exposes the server's participation query. These
additions let customer UI use the same working game transactions as the bundled interface.

`ConfigureExtractor(float, bool = true)` replaces its one-parameter signature. Raw-source placement
now requires a compatible, live, neutral deposit by default and exempts only that deposit from the
collision check. Ordinary one-argument C++ calls remain source-compatible. Self-contained custom
extractors must disable **Require Raw Source For Placement** in their defaults or pass `false` before
play. Exact one-parameter member-function pointer bindings must be updated and native consumers
rebuilt. Existing Blueprint nodes acquire the default-true input. No assets or types are renamed.
See [Economy](ECONOMY.md), [UI Data API](UI_DATA_API.md), and [Match State](MATCH_STATE.md).

Private helpers, lifecycle overrides, and squad state have no new customer callable surface.
The opt-in private technology acquisition diagnostic contributes a reflected class record only.
The reviewed compiled delta contains **35 added records, one changed function, and no removals**.
Header fingerprint → `ADD42287F5769E100F218492A37DB98B654E9C74` (**176 headers**);
reflected fingerprint → `E30A4723F3C621E177E60F157FA3E7EC4D7B77C8` (**2,009 records**).

---

## Battlefield-aware opponent choices — additive

**Date:** 2026-09-11 · **Stage:** pre-release

`ARTSSkirmishGameMode::GetSupportedOpponentCounts()` exposes the same battlefield bounds and
minimum-human-slot rules used by match setup, without changing the selected lineup. Native and
customer setup widgets can offer valid counts directly. The query returns no choices outside the
authoritative pre-start phase. Existing configuration and start methods keep their validation.

One const BlueprintPure array-returning function is added; no prior declarations change. Header
fingerprint → `E3AD9A54699A35E7AD80B94A384C899B2A50A1E6` (**173 headers**); reflected fingerprint
→ `5EBB11E5374154B30C6AEDF33C3EB2E849CA6CE4` (**1,974 records**). No migration is required.

---

## Faction field guidance — documentation metadata only

**Date:** 2026-09-11 · **Stage:** pre-release

Added customer tooltips to DisplayName, Description, and Crest in the runtime and authoring faction
structs. No fields, signatures, layout, or serialized values changed; no migration is required.
Removing exactly these six comments reproduces the preceding header fingerprint. Header fingerprint
→ `DB7770612E05FAA8145647A5F8D39347F320ACD5` (**173 headers**). The reflected contract remains
`5FFD8846B22470E9BA59E54DF69F63E23F82701A` (**1,973 records**).

---

## Reskinnable native setup and match clock — additive

**Date:** 2026-09-11 · **Stage:** pre-release

The new Blueprintable `URTSSkirmishSetupWidget` supplies a working native layout and supports
customer Widget Blueprint layouts through documented control bindings. `ARTSHUD` exposes a setup
widget class override and instance getter. Setup captures UI input and restores RTS controls after
successful start. Single-faction games omit redundant participant selectors.

`ARTSGameState::RestartMatchClock()` resets only the authoritative clock at successful skirmish
startup, so time spent choosing opponents is excluded. It does not clear elimination/result state
or broadcast a rematch. Existing clock/reset APIs retain their behavior.

Exactly four reflected records are added: the setup widget class, HUD class property and getter,
and clock method. No prior records change or disappear. Header fingerprint →
`57DC77E880B43E44E9673BE0AA6B32CFF2140A2F` (**173 headers**); reflected fingerprint →
`5FFD8846B22470E9BA59E54DF69F63E23F82701A` (**1,973 records**). Existing scenarios retain automatic
startup unless explicitly configured for player setup. No customer migration is required.

The guided clip-mapping dialog and asset-creation helper are editor-private and add no runtime
API. Saved Anim Sets retain the existing public data contract.

---

## Playable factions and explicit setup — additive, generator 6

**Date:** 2026-09-11 · **Stage:** pre-release

Authoring gains `FRTSStarterFactionDefinition` and an optional starter-scenario faction catalog.
The runtime receives resolved `FRTSSkirmishFactionDefinition` entries and participant selections.
Host-only configuration validates every catalog entry before applying a lineup or spawning actors;
invalid changes preserve the prior setup. Available/selected faction queries and an explicit
waiting-for-setup query support native or customer UI. Catalog-free existing scenarios preserve
their shared starting classes and automatic startup. Rematches retain the selected lineup.

The starter scenario struct moved to `Authoring/RTSContentSetScenario.h`; its reflected identity and
umbrella include remain intact. Source schema stays 2; generator 6 regenerates faction wiring.
The catalog array removes the compiler-derived `CPF_NoDestructor` flag from `StarterScenario`;
this is the expected lifetime consequence of owning an array, with no property type/name change.
There are exactly **25 added reflected records and one derived-flag change**: two structs,
18 properties, and five callable queries/configuration methods. No records disappear.

Return navigation also removes obsolete private approach-distance state and corrects the public
compatibility-method comment: all deposits use the authoritative interaction reach. No return
method signature changes; out-of-range calls retain cargo.

Reviewed foundation header fingerprint → `FB2264B3022EC463006B62D05AD060C1A43CD86E` (**172 headers**).
Reflected fingerprint → `BB53E520AB4196BE797D5324AF8B9FD764B5FCDB` (**1,969 records**).
No native/Blueprint source migration is required; regenerate owned output to adopt faction wiring.

---

## Native rematch flow and Python validation diagnostics — compatible/additive

**Date:** 2026-09-11 · **Stage:** pre-release

The existing match banner gains private result-reset and rematch handlers. Hosts can invoke the
existing authoritative skirmish reset; clients receive a waiting state. Reset and widget
reattachment clear stale result UI. No callable runtime signature or reflected runtime record
changes. Header fingerprint → `5740B6B2422D80971C8BB867C8F607E00D6BAC71` (171 headers);
reflected fingerprint remains `A962D5BEB0D037FA12B979D71FCF89E4A74C6F9E` (1,944 records).

Editor-only `GetContentSetValidationMessages(ContentSet, bOutValid)` returns messages directly
and validity separately, so Python callers retain diagnostics on invalid input. Unreal's Python
binding converts a false Boolean return to `None` and discards output parameters on the old
`ValidateContentSet` method. Existing Blueprint/C++ callers keep their original API; Python tools
can use `messages, valid = get_content_set_validation_messages(content_set)`. This editor-only
addition is outside the runtime freeze. No asset migration is required.

---

## Complete-game authoring recovery — breaking and additive, permitted pre-release

**Date:** 2026-09-11 · **Class:** breaking authoring repair representation and authority metadata;
additive progression, defense, and repair inspection · **Stage:** pre-release

| Change | Purpose | Customer migration |
| --- | --- | --- |
| Content Set schema 2 / generator 5; `FRTSUnitDefinition::Repair` now uses `FRTSRepairDefinition` with stable resource-id costs | Author repair before generated resource classes exist; retain and explicitly migrate old class-keyed costs | Use Preview/Apply Content Set Upgrade, resolve resource identities, then regenerate before cooking. Native authoring uses `FRTSRepairDefinition`; runtime `FRTSRepairData` remains available. |
| `RequiredBuildingIds` on units; building `Attacks`; research upgrade/building/per-level gates and cancellation refund | Express a complete tech tree and automatic defensive structures in the source ruleset | Additive empty catalogs preserve existing authored choices. Supply intended prerequisites and attacks explicitly. |
| `ConfigureRepair` is authority-only; `GetRepairData` exposes an immutable profile copy; native example worker includes repair | Prevent client-side configuration and support native/generated repair composition | Configure authoritative instances or templates. Client calls must be routed through an authorized server workflow. |
| Public economy/research headers extract existing amount/research structs and introduce repair and research-gate structs | Keep authoring contracts cohesive without changing existing reflected type identities | Existing `RTSContentSet.h` includes remain valid; narrower native includes are optional. No reflected type rename requires a redirect. |
| Shared supply/skirmish ceilings; private command-card paging and scoped input bindings; stricter configuration contracts | Keep validation aligned with runtime limits and preserve access to large command catalogs | No signature migration for these changes. Invalid configuration is rejected transactionally. |

Canonical reflection review accounts for exactly **18 added records and two changed records**.
Reversing those reviewed changes reconstructs all **1,926** previous records and their exact SHA-1
`FB260D15B28305A98378E971CBFD91B7601F8C98`; no unexplained reflected drift remains. The two changed
records are the repair property's struct type and `ConfigureRepair`'s authority flag. The additions
cover three structs and their fields, four research fields, unit prerequisites, building attacks,
the repair getter, and the example worker's repair component. Text-only/private header changes
were reviewed separately, including paging helpers and shared runtime constants.

Header fingerprint → `114772EACB56729C8E507D449403186A10B5B99E` (**171** headers).
Reflected fingerprint → `A962D5BEB0D037FA12B979D71FCF89E4A74C6F9E` (**1,944** records).
These are development recovery changes, not an installed-release qualification. Keep the ledger,
implementation, manifest, and reviewed fingerprints together when checkpointing this work.

---

## Distributed documentation and provenance references corrected — non-breaking

**Class:** non-breaking (comment-only public-header corrections) · **Stage:** pre-release

| | |
| --- | --- |
| What | Seven public headers now point to shipped customer guides instead of removed or repository-only planning files. The three extracted selection, camera, and order-submission component headers also state their retained ue4-rts PlayerController lineage and point to the shipped BOM/notices. Selection belongs to both sets, so nine unique public headers changed. No header path, declaration, signature, metadata specifier, or reflected record changed. |
| Why | A commercial source distribution must not send customers to files deliberately excluded from the package, and extracting inherited implementation into a new basename must not make its provenance disappear from the source. The byte-level native-header check intentionally notices comments, so the reviewed text correction requires an explicit rebaseline even though the reflected surface stays byte-identical. |
| Customer impact | None. Native and Blueprint integrations compile and behave identically. Customers gain valid in-package documentation references and clearer source-level attribution. |

Header fingerprint → `F9517D45050B6C7ABD18683352DD6D570E0112DC` (**169** headers).
Reflected fingerprint → `FB260D15B28305A98378E971CBFD91B7601F8C98` (**1,926** records, unchanged).

---

## Match restart and AI construction queue hardened — non-breaking

**Class:** non-breaking (runtime bug fixes; private declarations only) · **Stage:** pre-release

| | |
| --- | --- |
| What | In-place match reset now releases the framework-owned match-end pause before scheduling replacement-battlefield work. The built-in AI construction consumer now performs one bounded queue pass: a failed non-blocking placement/order attempt remains queued while later work is considered, whereas a failed blocking item still stops the pass. Urgent supply seeding occurs after ordinary standing intents so it cannot tie with a newly seeded extractor. The public-header path set remains 169; its text changed only for private queue helper declarations and conditional automation-test friendship. |
| Why | Match end pauses the UE world, so the previous next-tick reset continuation could never run. Separately, equal-priority insertion could place a transiently failing refinery ahead of an urgent Supply Depot forever: the consumer retried that one non-blocking item each tick and never reached the blocker behind it. Both failures were reproduced through production paths and now have focused automation coverage; the reset also passed two-match NullRHI and visible Metal-rendered qualification. |
| Customer impact | No source or Blueprint migration is required. Intentional in-place restart resumes simulation instead of remaining paused. Built-in AI retains transiently rejected non-blocking construction intent but no longer lets it starve later urgent work. Blocking queue semantics are unchanged. |

Header fingerprint → `47374552C1C99647342C7B282D01B8E435A209CF` (**169** headers).
Reflected fingerprint → `FB260D15B28305A98378E971CBFD91B7601F8C98` (**1,926** records, unchanged).

---

## MatchCheck transaction probe extracted; dead capture API removed — breaking, permitted pre-release

**Class:** breaking (exported native signatures narrowed and an overload removed) ·
**Stage:** pre-release

| | |
| --- | --- |
| What | The complete opt-in customer-transaction proof moved from `URTSMatchCheckSubsystem` into the private, non-`UObject` `FRTSMatchCheckTransactionProbe` and five private source files. The subsystem remains the only world subsystem, tick owner, reflected delegate target, command-line/config owner, alarm sink, and verdict publisher; its three transaction callbacks retain their exact reflected signatures. Separately, the exported `FRTSMatchPerformanceCapture::Initialize` and `RecordPopulation` functions no longer accept the unused `ProcessPeakPhysicalBytes` parameter, and the lossy duration-only `RecordGarbageCollection(double)` overload was deleted. No public-header path was added or removed (still 169). |
| Why | Measurement found no dead MatchCheck state machine to delete: transaction readiness was the first cohesive state-plus-mechanism boundary. Moving its 44-field state, all seven active phase branches, temporary qualification advantage, aggregate completion count, and exact controller/wallet/builder delegate ownership reduced the main implementation from 5,829 to 4,603 lines without adding a second tick or reflected surface. The extraction also exposed inherited lifecycle gaps: every terminal path now restores the temporary advantage and unbinds every tracked delegate immediately, including timeout and early verdict finalization, so late callbacks cannot mutate sealed evidence. The removed capture parameters were read nowhere, while the deleted GC overload fabricated a timestamp and discarded post-collection memory provenance. |
| Customer impact | Native callers must remove the process-peak argument from both capture functions. For `Initialize`, the old sixth `UnrealAllocatedBytes` argument becomes the fifth; for `RecordPopulation`, every argument after `UsedPhysicalBytes` moves left by one. This migration must be explicit because the all-numeric signatures can otherwise compile while silently rebinding process-peak bytes as Unreal allocator bytes (and allocator bytes as `Components`). Callers of `RecordGarbageCollection(DurationMs)` must use the retained six-argument overload and provide wall timestamp, explicit/natural provenance, post-resident bytes, post-Unreal bytes, and availability. Recompile native integrations; no Blueprint, serialized-asset, redirect, or public callback migration is required. |

Header fingerprint → `71998D8B1AD25843751D3452CADBDB792B4788C6` (**169** headers).
Reflected fingerprint → `FB260D15B28305A98378E971CBFD91B7601F8C98` (**1,926** records, unchanged).

The header hash moves for two separately reviewed reasons: the private transaction pimpl reshapes the
advertised subsystem header, and two unused capture parameters plus one lossy overload leave the C++ contract.
The reflected hash remaining byte-identical proves that the subsystem's Blueprint/delegate contract,
callback owners, property metadata, and verdict-facing reflected API did not move with the mechanism.

---

## Order submission moved to URTSOrderSubmissionComponent — breaking, permitted pre-release

**Class:** breaking (public members and RPC ownership relocated) · **Stage:** pre-release

| | |
| --- | --- |
| What | Selected-unit submission, structural request validation, strategic-request counters, packet-safe batch state, client flow control, and order transport moved from `ARTSPlayerController` to a new `URTSOrderSubmissionComponent`, reached through `ARTSPlayerController::GetOrderSubmission()`. Moved public functions: `IssueOrderToSelectedActors`, `IsClientOrderRequestStructurallyValid`, `IsClientOrderBatchStructurallyValid`, `GetAcceptedStrategicRequestWorkUnits`, and `GetRejectedStrategicRequestWorkUnits`. The `ServerIssueOrder`, `ServerBeginOrderBatch`, `ServerAppendOrderBatchChunk`, `ServerIssueOrderBatch`, and `ClientFinalizeOrderBatch` RPCs moved with their validation and batching mechanism. New public header `Orders/RTSOrderSubmissionComponent.h` (168 → 169 headers). |
| Why | The controller was still carrying a complete secure transport subsystem. The new component is a replicated native default subobject with the stable, owning-client identity required for component RPCs. It owns the one shared strategic-request limiter: the controller's retained ability, research, stance, unload, construction, production, and surrender RPCs consume that same bucket rather than creating a second admission path. A server transaction expires after five wall-clock seconds; the sole owning-client large-batch slot reports a missing acknowledgement after ten seconds but stays occupied until its reliable reply arrives. Those states enable an otherwise-idle component tick only while timeout work exists. The controller wires its exact native selection component plus one-way policy, execution, queue-modifier, error, confirmation, and prediction delegates. |
| Customer impact | Native `PC->IssueOrderToSelectedActors(Order)` becomes `PC->GetOrderSubmission()->IssueOrderToSelectedActors(Order)`; the two validators and two counter getters migrate the same way and native callers include `Orders/RTSOrderSubmissionComponent.h`. Blueprint callers retarget those nodes through `GetOrderSubmission`. Command-routing APIs remain on the controller: `IssueDefaultOrderToActor` and the attack, move, gather, construction, production, ability, research, cancellation, stance, unload, and stop entry points do not move. `IsOrderClassAllowedFromClient`, `DefaultOrders`, `DefaultOrderIgnoreTargetClasses`, `AddDefaultOrderClass`, `bAddSelectionHotkeyPressed`, controller events, and non-order RPCs also remain in place. No serialized asset in either `Content/` tree referenced a moved symbol, so the repository requires no Blueprint asset migration. |

Header fingerprint → `13128F2518969F83C09A361A0487E12F2A3BB89A` (168 → **169** headers).
Reflected fingerprint → `FB260D15B28305A98378E971CBFD91B7601F8C98` (1,924 → **1,926** records).

The reflected count rises by exactly two: one class record for `URTSOrderSubmissionComponent` and one
function record for `GetOrderSubmission()`. The five public functions and two editable limiter
settings changed owner without changing the record count. The freeze excludes the private RPCs, so
their owners, validation, direction, and reliability are asserted separately by reflection tests.

Focused real bidirectional PIE qualification resolves distinct owning-client and authority component
instances, verifies their stable replicated default-subobject identity, sends a generated
client-to-server component RPC with a 32-reference pawn array, observes authority admission and
independent timeout maintenance, and receives the real reliable `ClientFinalizeOrderBatch` response
on the owning client without closing either connection. The 32 references deliberately reuse one
already-mapped pawn, so this does **not** prove the worst-case first export of 32 distinct NetGUIDs.
The in-editor listen-server proof also does not replace packaged hostile-traffic or saturation
qualification; those remain explicit release-level limits rather than claims made by this change.

---

## Camera control moved to URTSCameraControlComponent — breaking, permitted pre-release

**Class:** breaking (public members relocated; two dead native templates removed) · **Stage:** pre-release

| | |
| --- | --- |
| What | Camera focus, center, selection-follow, bounds, and local bookmark state moved from `ARTSPlayerController` to a new `URTSCameraControlComponent`, reached through `ARTSPlayerController::GetCameraControl()`. Moved with unchanged reflected signatures and metadata: `IsFollowingSelection`, `GetFollowedActor`, `ToggleFollowSelection`, `CenterCameraOnSelection`, the three `FocusCameraOn*` functions, and `SaveCameraLocation` / `LoadCameraLocation`. New public header `Camera/RTSCameraControlComponent.h` (167 → 168 headers). The zero-use, unreflected `SaveCameraLocationWithIndex<N>` and `LoadCameraLocationWithIndex<N>` templates were deleted instead of being relocated. |
| Why | Camera was the remaining half of the controller/selection dependency cycle. It now reads selection through the already-extracted `URTSSelectionComponent`, while the controller keeps viewport input, pan/zoom/drag state, tick ordering, and its Blueprint event hub. A controller-called follow seam preserves the material order `pan input → follow/manual-break → bounds`; independent component ticking would have changed behavior. Measuring first also found the two dead templates, so deletion was smaller and more honest than moving unused API. |
| Customer impact | `PC->FocusCameraOnActor(Target)` becomes `PC->GetCameraControl()->FocusCameraOnActor(Target)`, and likewise for the other moved functions. Native callers of either deleted `*CameraLocationWithIndex<N>` template must call the index-taking component function. `ARTSPlayerController::OnFollowSelectionChanged` remains in place, so existing Blueprint/UI event bindings do not move. No serialized Blueprint graph referenced a moved function: the only asset-string hits were the bundled Toggle Follow Input Action's own name in that asset and its mapping context. |

Header fingerprint → `7EC864C66D05D809C90CF1BC2466092ABED2B2BF` (**168** headers).
Reflected fingerprint → `E91E7D360E0A1004778822C9A11EE5AD51897A5D` (1,922 → **1,924** records).

The reflected count rises by exactly two: one class record for `URTSCameraControlComponent` and one
function record for `GetCameraControl()`. The nine reflected camera methods and
`CameraFollowInterpSpeed` only changed owner, while the controller's assignable follow event stayed
put.

The bookmark gate was preserved deliberately: bookmark arrays remain empty until
`SetupInputComponent`, repeated setup with the same count retains saved locations, and automation now
possesses a real camera pawn so the pre-setup rejection cannot pass for the unrelated reason “no
pawn.” The same test covers a bookmark round trip, follow interpolation, and the same-frame
manual-pan break.

---

## Selection moved to URTSSelectionComponent — breaking, permitted pre-release

**Class:** breaking (public members relocated) · **Stage:** pre-release

| | |
| --- | --- |
| What | Selection state and mechanics moved off `ARTSPlayerController` into a new `URTSSelectionComponent`, reached via `ARTSPlayerController::GetSelection()`. Moved: `SelectActors`, `GetSelectedActors`, `IsSelectableActor`, the subgroup family, and the control-group family. New public header `Player/RTSSelectionComponent.h` — the first path added since the freeze (166 → 167, index 120). |
| Why | The controller owned selection state, the camera read it back, and `SelectActors` called `CancelTargeting` and `FocusCameraOnActor` directly — a dependency cycle. The component now owns state and mechanism, and publishes `OnSelectionWillChange` / `OnSelectionChanged` / `OnSelectedSubgroupChanged` / `OnRepeatSelection` / `EmptySelectionHandler` so the controller keeps the cross-cluster policy. Selection depends on nothing. |
| Customer impact | `PC->GetSelectedActors()` becomes `PC->GetSelection()->GetSelectedActors()`, and likewise for the moved functions. No serialized asset referenced any of them (checked across all `Content/` before the move), so no Blueprint breaks. |

Header fingerprint → `8B404CAACE171E295FE85B4F6BFE2334522F5518` (**167** headers).
Reflected fingerprint → `24BA3261EC5B49AADB2F322A14DE593F6A1FC8B0` (1,917 → **1,922** records).

The reflected count rises by five rather than staying flat: the component contributes its own class
record, plus `GetSelection()`, `InitializeControlGroups`, and the two accessors that had no
controller equivalent. Every moved function kept its reflected shape.

**A real behaviour change was caught by the suite and reverted, not rationalised.** Sizing
`ControlGroups` in the component constructor made control groups addressable immediately, breaking
`RTS.Networking.InboundRequestSecurity`, which asserts they *cannot* be indexed before local input
setup. The original code sized them during local-player setup, gating them deliberately. Restored as
an explicit `InitializeControlGroups()` call from that same point, so the security property is now
named instead of incidental.

---

## Removed 20 dead control-group wrappers — breaking, permitted pre-release

**Class:** breaking (public `BlueprintCallable` removal) · **Stage:** pre-release

| | |
| --- | --- |
| What | `ARTSPlayerController::SaveControlGroup0()`–`SaveControlGroup9()` and `LoadControlGroup0()`–`LoadControlGroup9()` deleted. The index-taking `SaveControlGroup(int32)` / `LoadControlGroup(int32)` remain and are unchanged. |
| Why | Dead. Every reference in the repository was the declaration and its own definition. Enhanced Input binds `OnSetControlGroup`/`OnRecallControlGroup`/`OnAppendControlGroup` with the group index as a bound payload, so the numbered wrappers were never on any path. They were 20 of the controller's 111 public members — 18% of its public surface existing for nothing. |
| Customer impact | A Blueprint calling `SaveControlGroup3()` would break and must call `SaveControlGroup(3)`. Zero serialized assets in this repository reference them (checked across all `Content/` before removal), and no listing exists, so real-world impact is nil. |

Header fingerprint → `0A27AE56374ABD71F7E59E01CF71AF6950D32671` (still 166 headers — no path added or removed).
Reflected fingerprint → `6E3ED5FD36323AC01F5B6232A70ECD8D5FB553A3` (1,937 → **1,917** records).

**Both** fingerprints moved, which is the signature of a real contract change — contrast the formatting
sweep below, where only the header hash moved. The reflected count fell by exactly 20, one per deleted
`UFUNCTION`: that exactness is the evidence nothing else left the contract alongside them.

---

## Formatting sweep — header text only, contract unchanged

**Commit:** `20d53aa` · **Class:** compatible · **Stage:** pre-release

| | |
| --- | --- |
| What | Tree-wide clang-format application across 489 files. No declaration added, removed, renamed, or re-signed. |
| Why | Tabs and spaces coexisted inside single files (`RTSPlayerController.cpp`: 1,676 vs 971 lines), so the tree adopted one mechanically enforced formatting standard. |
| Customer impact | None. No recompilation required beyond the normal engine rebuild. |

Header fingerprint → `0078FC5CBB7A1DE23E41693D99014A2D4DEDE293` (166 headers).
Reflected fingerprint **unchanged** at `FDEFEBD3BA8C07FF1673F445C036F3E04CEB1032` (1,937 records).

This is the canonical example of the header/reflected split doing its job: the header hash moved
because header *text* moved, while the contract provably did not. Had the reflected hash also moved,
the sweep would have been reverted rather than re-baselined.

---

## Shift-queued orders — additive

**Commit:** `ac2c425` · **Class:** additive · **Stage:** pre-release

| | |
| --- | --- |
| What | `FRTSOrderData::bQueued`; `URTSOrder::bCanBeQueued` + `GetCanBeQueued()`; `ARTSPawnAIController::MaxQueuedOrders` and five queue entry points (`EnqueueOrder`, `AdvanceOrderQueue`, `ClearOrderQueue`, `GetQueuedOrderCount`, `GetQueuedOrders`). |
| Why | Shift+click waypoint queuing, a baseline Brood War control expectation. No existing member changed meaning. |
| Customer impact | None for existing integrations. A customer's Blueprint order subclass gains an authorable `bCanBeQueued` defaulting to `true`, matching prior behaviour for every non-queued order. |

Header fingerprint → `A813BE2D…` (166 headers, no new path).
Reflected fingerprint → `FDEFEBD3…` (1,928 → 1,937 records).

---

## v1 baseline

**Commit:** pre-`ac2c425` · Header `783E1EAC…` · Reflected `98CDFF3B…` (1,928 records).

Recorded for continuity; predates this ledger.
