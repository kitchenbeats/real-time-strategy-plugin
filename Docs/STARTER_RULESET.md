# Starter Ruleset and Reskin Workflow

The supported customer workflow is Unreal-native. It does not require StarMaps content, Python
scripts, or a particular character vendor.

## Clean-project contract

A blank Unreal 5.8 project can enable the plugin and choose **Add > Gameplay > RTS > RTS Content
Set** in the Content Browser. No generic Data Asset class picker or setup utility is required.
Every newly created Content Set starts with a valid greybox economy/combat ruleset: minerals, a
gatherable mineral field, a worker, a rifleman, a town hall, a barracks, and a supply depot. It can
immediately generate component-complete RTS Blueprints. Greybox units, buildings, and resource
sources use plugin-owned materials plus Engine basic shapes, so assigned art is optional.

## Version identity

Every newly saved `URTSContentSet` writes the plugin's dedicated Unreal custom-version GUID and the
current customer-visible `SchemaVersion`. The editor generator has an independent monotonically
increasing generator version. An asset loaded from before this contract is marked schema/custom
version 0; it is never silently relabeled as current merely because the plugin can deserialize it.
This identity is the input to the migration workflow. Until the migration gates in the commercial
release tracker pass, version detection alone is not evidence that updating production assets is safe.

Schema versions describe authored Content Set data. Generator versions describe the layout and
meaning of generated base assets. They change independently so an implementation-only generator
update does not pretend the customer's source asset schema changed, and a schema migration can
report exactly whether regeneration is also required.

## Generated ownership and extension boundary

Generation creates `DA_RTSGenerationManifest_<ContentSetId>` beside the category folders. It
records the source Content Set path/id, schema and generator versions, a unique completed-run id,
and the exact path, stable id, and semantic role of every generator-owned base asset. Regeneration
updates an existing output only when that manifest proves the exact ownership tuple. A compatible
Blueprint at the expected path is not presumed safe to overwrite; missing or mismatched provenance
fails before generation changes any assets.

Treat the manifest and every asset it lists as generated base implementation. Put customer-owned
logic and presentation in child Blueprints or composed classes outside the generated output root,
then reference those complete classes through `ExistingUnitActorClass`,
`ExistingBuildingActorClass`, or the equivalent resource/source fields. The generator registers
those referenced classes but never rewrites them. Do not add customer graphs, components, variables,
or defaults directly to a manifest-owned base: regeneration intentionally owns that complete asset.
See `CONTENT_SET_MIGRATION.md` for the dry-run fields, anomaly classifications, and blocking rules.

The generated gameplay layer includes ownership, selection, visibility, health, vision, movement,
AI defaults, attacks, gathering, construction, production, costs, supply, UI components, collision,
navigation footprints, and gameplay tags as requested by the definitions.

All framework gameplay tags are registered natively by the runtime module. Customers do not need
to copy a GameplayTags config section or reference the plugin's historical tag-table asset for
orders, gathering, construction, relationships, and status requirements to work.

By default, generation also creates a runnable starter scenario in the Content Set's `MapsPath`:

- `DA_Skirmish_<ContentSetId>` wires the generated town hall, worker, resource source, starting
  resources, human-controlled bases, AI fill, and workers. `MinimumHumanPlayersToStart` can hold
  startup until the intended number of humans has joined. Disable
  `StarterScenario.bHumanControlsFirstBase` for an AI-vs-AI observer showcase instead.
- `BP_GM_<ContentSetId>_Starter` exposes a Blueprint extension point while supplying the complete
  skirmish runtime.
- `L_<ContentSetId>_Starter` uses that GameMode and contains a collision floor, built Recast
  navigation, matching minimap and vision bounds, a configured fog-of-war controller and unbound
  post-process renderer, an RTS player start, and basic lighting.
- `ARTSHUD` creates the plugin-owned full-screen RTS console for each local player. Its native
  widget tree includes the resource and match strips, control groups, info panel, command card,
  caster bar, notifications, match banner, and the bundled minimap—no project UI Blueprint is
  required.

Open that map and press Play to exercise the generated ruleset. Disable `StarterScenario.bGenerate`
for a definitions-only Content Set, or change its IDs and layout settings before generation to pick
different generated building, worker, and resource-source classes.

The commercial automation suite also opens the bundled reference in PIE and completes a real economy
vertical slice through public gameplay APIs: match startup, participant/team creation, generated actor
counts, worker selection, gather-order routing, navigation, extraction, and deposit into the player's
resource wallet. This runtime test is repeated against the isolated installed package during release.

## Bundled playable reference

Customers do not have to generate anything to confirm the plugin is installed correctly. Enable
**Show Plugin Content**, open
`/RealTimeStrategy/Starter/Reference/Generated/Maps/L_RTSStarter_Starter`, and press Play. This
plugin-owned map is a complete two-base skirmish with built navigation, a flat collision floor,
resources, workers, production buildings, combat units, fog of war, the default HUD, and the native
AI runtime.

`/RealTimeStrategy/Starter/Reference/DA_RTSStarter` is the reproducible source Content Set for the
reference. Its generated assets are intentionally grouped beneath
`/RealTimeStrategy/Starter/Reference/Generated/{Resources,Units,Buildings,Maps}` so the supported
example is visually separate from framework assets and from customer-owned `/Game` content.

The bundled actors use redistributable, vendor-neutral greybox presentation. This is the guaranteed
blank-project baseline, not a skeleton limitation: assign or retarget Manny, Mixamo, Meshy,
marketplace, or custom studio assets through the reskin contract below. The plugin never requires or
hard-references optional project/vendor content.

## Fastest Blueprint-only character reskin: UE 5.8 Manny

Use this path when the project was created from UE 5.8's Third Person template or has the matching
**Characters** feature pack. Manny remains customer/Epic-owned project content; the plugin discovers
it in `/Game` and never redistributes or hard-references it from plugin content.

1. Create an **RTS Content Set** and keep its starter worker plus at least one unit whose Role is
   `Combat`.
2. Select the Content Set in the Content Browser, right-click it, and choose **RTS Authoring > Apply
   UE 5.8 Manny Presentation**.
3. Review the **RTS Authoring** Message Log. The action validates the exact UE 5.8 mesh, Animation
   Blueprint, locomotion, labor, attack, and death assets before changing the Content Set. If the
   required content is absent, add the UE 5.8 Characters feature pack and run the action again.
4. The action creates or updates customer-owned worker and combat `RTSAnimSet` assets under
   `<ContentSetFolder>/Presentation/Manny`, assigns Manny and the compatible custom Animation
   Blueprint to those unit definitions, saves the animation sets, and marks the Content Set dirty.
5. Choose **Validate Content Set**, then **Generate Playable RTS Content**, open the generated
   `L_<ContentSetId>_Starter` map, and press Play.

The operation is undoable and repeatable. It refuses package/type conflicts and keeps presentation
assets outside the generator-owned output tree, so later regeneration does not overwrite them. Use
**Apply Imported Unit Presentation...** instead for Mixamo, Meshy, marketplace, or studio-owned
meshes; that vendor-neutral path is documented in `Docs/MULTIRIG_ANIM.md`.

## Blueprint workflow

1. In the Content Browser, choose **Add > Gameplay > RTS > RTS Content Set**.
2. Use the included starter definitions, edit them, or press `Reset to Starter Ruleset` to restore
   the supported baseline.
3. Add resources, resource sources, units, and buildings in the Details panel as needed.
4. Leave `VisualMode` on `UseGreyboxPrimitive` for an immediately playable prototype, or assign
   project art and choose `UseAssignedArt`.
5. Right-click the Content Set in the Content Browser and choose **Validate Content Set**, then
   **Generate Playable RTS Content**. **Apply Safe Authoring Fixes** repairs deterministic issues
   in one undoable transaction and reports any design decisions that still need attention.
6. Open `L_<ContentSetId>_Starter` and press Play. Reskin generated units by changing their
   presentation fields on the Content Set and regenerating. For game-specific Blueprint behavior,
   either keep a customer-owned child of the generated base outside the generated output root and
   select it from a customer-owned roster/map/configuration seam, or supply a complete stable
   project-owned class through the Content Set's `Existing*Class` field. An `Existing*Class` must
   not depend on the generated class it suppresses. Never edit a manifest-owned generated base.

Every authoring structure is `BlueprintType`, its customer fields are `BlueprintReadWrite`, and the
editor library reports guided validation findings instead of silently generating broken assets.
All three native Content Browser actions support multi-selection and publish detailed results to
the **RTS Authoring** Message Log. Editor Utility Blueprints can call the same
`RTSContentSetEditorLibrary` functions when a studio wants a custom tool surface.

Commercial release candidates use the timed, unassisted human study in
`UI_AUTHORING_QUALIFICATION.md`. Automated generation proves correctness but never substitutes for
that discoverability and ergonomics evidence.

Before changing any output, generation preloads and preflights every expected resource, source,
unit, building, starter definition, GameMode, and map path. An incompatible existing asset fails the
operation with its exact path and type before partial output is created. Generated Blueprints must
compile without errors or warnings before the deferred package save begins. When saving is enabled,
the generator also validates every final filename, attempts source-control checkout for read-only
existing outputs, and proves every destination directory with a real temporary write before mutating
assets. Packages are then saved in deterministic path order. This protects normal validation,
collision, compiler, permission, checkout, and directory failures; it is not a claim of
filesystem-atomic rollback if the operating system or source-control provider fails unexpectedly
partway through Unreal's sequential multi-package save.

Generated package names and category folders are deterministic. Automation saves a real Content Set
and rejects missing, stray, misnamed, or redirector outputs. Generated project assets may depend on
customer-selected `/Game` presentation assets—that is how reskinning works—but the bundled plugin is
separately required to have no `/Game` dependency. The saved automation fixture also proves that
generated cross-package dependencies remain inside its selected output root; custom art outside
that root is allowed only when the Content Set explicitly selects it.

Blueprint UI authors can derive from `ARTSHUD` and `URTSConsoleWidget`, replace the console or
minimap classes in Class Defaults, replace individual console panel classes, and use the
`Console Widget Created` event for game-specific bindings. Set `Auto Create Console Widget` off
only when a project supplies a wholly custom root HUD.

The plugin selects its bundled `DA_RTSHudStyle_Default` skin without requiring a project config
entry. A project can replace it under **Project Settings > Game > RTS HUD Style**, or set a
per-GameMode style override on its `ARTSHUD` subclass. If an authored style is deliberately unset,
the native style defaults remain a complete asset-free safety net.

## C++ and command-line workflow

Code projects use the same `URTSContentSet`, `FRTSUnitDefinition`,
`FRTSBuildingDefinition`, validator, and generator backend. Headless validation and generation are
available for CI:

```text
UnrealEditor-Cmd Project.uproject -run=RTSValidateContentSet -ContentSet=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -ContentSet=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -ContentSet=/Game/MyRTS/DA_MyRules -ApplyUpgrade -OutputRoot=/Game/MyRTS/Generated
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MyRules -OutputRoot=/Game/MyRTS/Generated
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_NetRules -OutputRoot=/Game/MyRTS/Generated -MinHumanPlayers=2
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MannyRules -OutputRoot=/Game/MyRTS/Generated -Presentation=Manny -Observer
```

`-CreateStarter` creates and saves a complete starter Content Set before running the same
production generator used by the editor actions. Without `-OutputRoot`, generated categories are
placed in a `Generated` folder beside the new Content Set. Creation refuses to replace an existing
asset. Pass `-Force` only when the asset is already an `RTSContentSet` and CI intentionally needs to
reset it to the supported starter ruleset and regenerate its known output assets. Assets of another
class are never overwritten, invalid mounted paths fail before generation, and every failure returns
a nonzero process status.

For an existing Content Set, `-OutputRoot` relocates only its generated category paths and preserves
its stable Content Set id. This supports project-owned migration and CI staging without writing into
the source plugin. Normal provenance and ownership checks still apply, so relocation cannot silently
claim or overwrite unowned assets.

`-Presentation=Manny` discovers the supported assets from an installed UE 5.8 Third Person template
and authors project-owned worker/soldier Blueprints plus customer-owned animation sets under
`<ContentSetFolder>/Presentation/Manny`; `-OutputRoot` affects only generated output, and the plugin
retains no `/Game` dependency. It assigns the template's compatible Manny Animation Blueprint so
the feature pack's root-motion-tagged clips are evaluated without surrendering authoritative RTS
navigation. `-Observer` makes the generated starter map an autonomous AI-versus-AI
showcase. `-MinHumanPlayers=N` sets the multiplayer startup gate and must not exceed the starter's
base count; observer scenarios require zero. Connected humans are assigned deterministically to the
first available bases, remaining bases use AI, excess and post-start joins become observers. To
package that map as the application entry point, explicitly select its generated
`L_<ContentSet>_Starter` map under **Project Settings > Maps & Modes > Game Default Map** before
cooking. Generation deliberately does not rewrite project-wide startup configuration.

For an automated packaged human-side acceptance run, generate the Manny profile without
`-Observer`, select the generated map as `GameDefaultMap`, and launch the packaged executable with:

```text
-RTSMatchCheck=900 -RTSMatchCheckRepeats=2 -RTSMatchCheckSpeed=16 -RTSHumanCheck -RTSTransactionCheck
```

The structured
verdict requires ownership, camera movement through the bundled Enhanced Input action, HUD and
team vision, selection, gathering, resource return, match completion, and reset in each match.

Runtime systems remain ordinary exported Unreal classes and components, so a code-only project can
derive its own controllers, orders, AI systems, components, widgets, and gameplay policies without
using generated Blueprints.

Content Sets also accept complete hand-authored actor classes. Set `ExistingUnitActorClass` on a
unit definition to a Character Blueprint or native `ACharacter` subclass, or set
`ExistingBuildingActorClass` to a Pawn Blueprint or native `APawn` subclass. The generator registers
that exact class in production, construction, starter-scenario, and skirmish rosters and does not
create or rewrite `BP_<Id>` for the entry. This is the code-first route for studios whose gameplay
configuration already lives in constructors or class defaults.

An existing class is authoritative: its components own visuals, combat/gather/build catalogs,
costs, production/drain catalogs, and presentation. The definition still owns the stable id,
faction/role, and scenario roster references. **Validate Content Set** checks the complete shared
RTS component contract plus every component implied by the entry's declared capabilities and gives
an actionable finding for each omission. For builders and production buildings, the existing
class must already reference the intended constructible/product classes; generation deliberately
does not mutate customer-owned classes. Clear the existing-class field whenever the definition
should generate and configure a Blueprint instead.

Code projects can derive from `ARTSHUD`, override `HandleConsoleWidgetCreated`, or replace
`ConsoleWidgetClass` and `DefaultMinimapWidgetClass`. `GetConsoleWidget()` exposes the live root
without requiring viewport searches or project-specific globals.

Blueprint and C++ projects can implement custom orders without changing plugin source. Blueprint
orders override the native events on `RTSOrder`; C++ orders override the matching `_Implementation`
methods. See `Docs/ORDERS.md` for registration, command-card, networking, and authority details.

Economy components expose the same two-language extension contract for custom harvesting,
resource-source admission, deposits, affordability, and payments. See `Docs/ECONOMY.md` for the
authority and data-invariant requirements.

Builder, construction-site, and production components expose Blueprint Native Events for their
core policies and transitions. See `Docs/CONSTRUCTION_PRODUCTION.md` for queue, payment, refund,
rally-point, and authority rules.

Combat actions from widgets route through the owning player controller, and combat state remains
server-authoritative and replicated. See `Docs/COMBAT_NETWORKING.md` for custom Blueprint/C++
ability targeting, stance, attack, research, and mutation rules.

## Built-in QA spawning

`ARTSPlayerController` uses `URTSCheatManager` by default, so PIE sessions and other sessions in
which Unreal permits cheats can spawn deterministic inspection units without project code. The
console commands `SpawnInspect Worker` and `SpawnInspect Rifleman` resolve plugin-owned starter
classes. `SpawnInspect` also accepts a full soft class object path ending in `_C`.

Blueprint projects can derive a Cheat Manager Blueprint from `URTSCheatManager`, add their own
short names to **Inspectable Unit Classes**, and select that class on the Player Controller.
C++ and Blueprint code can bypass string lookup with `SpawnInspectClass`, or use
`ResolveInspectableUnitClass` when they want the same alias/full-path resolution. Spawning is
server-authoritative; an inspection unit belongs to the first AI player when one exists and falls
back to the invoking player otherwise. Unreal disables cheat managers in Shipping builds by
default, so this QA surface does not create a production-game command channel.

## Reskin contract

Gameplay size comes from authored radius, height, and footprint data—not mesh bounds. Replacing art
therefore does not silently change selection, pathing, construction placement, attack range, gather
reach, or navigation blocking.

Static units may assign `StaticMesh`. Skeletal units assign:

- `SkeletalMesh`
- a skeleton-compatible `RTSAnimSet` containing locomotion and action clips
- `Direct Anim Set` for the zero-AnimBlueprint path, or `Custom Animation Blueprint` for an
  advanced graph that consumes `RTSAnimComponent` state

Editor Utility Blueprints and C++ authoring tools can call `ValidateImportedUnitPresentation`
before changing a Content Set. It runs the same read-only structural checks as the native apply
workflow and returns stable coded findings for CI or batch-import routing. Passing that preflight is
necessary but does not replace rendered retarget, deformation, and camera-distance review; use the
fixture evidence checklist in `MULTIRIG_ANIM.md` before claiming a specific rig family.

This is vendor-neutral. Manny, Mixamo, Meshy, marketplace characters, and studio-owned rigs all use
the same contract after their clips are imported or retargeted to the selected skeleton. The plugin
does not bundle or hard-reference project-specific vendor assets.

## Product boundaries

- Bundled plugin assets must never depend on `/Game` packages.
- StarMaps faction data, maps, meshes, animation examples, and tuning remain project content.
- Experimental StarMaps authoring scripts are not part of the customer workflow or verification.
- Customer documentation and runtime/editor modules are included by `BuildPlugin`.
- Clean packaging, bundled-content isolation, Blueprint generation, and runtime contracts are
  enforced by automation tests.
