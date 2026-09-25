# Starter Game and Generated Content

This is the reference for the included starter game, what **Generate Game** creates, and how to
generate from the command line. For everyday editing, start with [The Content Set](CONTENT_SET_GUIDE.md).
Nothing here needs Python scripts, content from another project, or a particular character vendor.

## The installed starter game

**RTS > Install the Starter Game** (also in **Tools > Real-Time Strategy**) copies the plugin's
starter Content Set into `/Game/RTSStarterGame`, generates your project's own copy of the game, and
opens its map. In a fresh project it also sets the game and editor startup maps; startup maps you
already chose are kept.

The starter game has one playable faction, **Expedition**, with eight units and eight buildings. It
covers minerals and gas, workers, construction, supply, research, infantry, armor, aircraft, siege,
healing and defense towers. Its art uses only engine basic shapes and plugin placeholder materials.

Your Content Set at `/Game/RTSStarterGame/DA_RTSStarter` is the editable source of your game.
Running **Install the Starter Game** again keeps it and its rules; it does not overwrite your
settings from the plugin's copy. To install another copy in a different folder, call the
**Install the Starter Game** node (`RTS|Authoring|Onboarding`) from an Editor Utility Blueprint and
pass a different `/Game` folder.

The plugin also contains a smaller sample, `/RealTimeStrategy/Starter/Reference`, used as a minimal
example and for automated tests. A Content Set you create with **+ Add > Gameplay > RTS Content Set**
starts with a similar small ruleset.

## Twin Crossings and player count

The starter game's map, **Twin Crossings**, is for two players: one human against the AI, or two
humans against each other. Extra network connections join as spectators. They do not take a player
slot and do not become players on a rematch.

Twin Crossings is a symmetric 120 by 120 meter map with a main base on each side and expansion
resource lines to the north and south. A low central island splits the ground into two wide routes.
The island blocks ground units and gives no height or cover bonus; aircraft fly over it normally.
Each main base has eight mineral fields and one gas geyser; each expansion has six mineral fields
and one gas geyser.

**Match Setup > Arena Preset** in the Content Set chooses **Twin Crossings** or **Open Arena**.
Twin Crossings needs **Num Players** 2, **Maximum Participants** 2 and a **Playable Half Extent**
of 6000 cm. **Open Arena** is a flat map with bases in a circle and allows up to sixteen players,
limited by the map size. The same limit applies to the setup screen, server travel options, lobby
requests and rematches; a request that does not fit fails before the map is set up.

Matches started without the setup screen (the `rts.skirmish.start` console command, a dedicated
server, an automated check) read these travel options on the map URL: `?NumBases=` (players,
human and AI), `?StartingWorkers=`, `?MinHumanPlayers=`, and `?AIDifficulty=Passive`, `Easy` or
`Normal` (default `Normal`). For example
`L_RTSComplete_Starter?AIDifficulty=Easy`.

The island is made of engine cube meshes with a map-owned copy of the plugin's placeholder material,
plus a volume that removes the same area from the navigation mesh. For your own terrain, work in
your own copy of the map instead of editing generated actors, which Generate replaces.

The map also stores a small terrain picture for the minimap, showing only the ground (no units or
resources). Live minimap markers still respect fog of war. A background brush set on your own
minimap widget replaces this picture.

## What Generate creates

Generate writes into the Content Set's output folders (**Advanced > Generated Content Root** and
**Output Paths**):

- `BP_<Id>` for every unit and building, `BP_Res_<Id>` for every resource and `BP_Src_<Id>` for
  every resource source. Each Blueprint contains the RTS components its Content Set entry needs:
  ownership, selection, visibility, health, vision, movement, AI, attacks, gathering, construction,
  production, costs, supply, health and progress bars, collision, navigation footprint and gameplay
  tags.
- `DA_Skirmish_<ContentSetId>`, the match settings: starting buildings, workers, resources, human
  and AI players. **Minimum Human Players to Start** can hold the match until enough humans join;
  turn off **Human Controls First Base** for an AI-only match you watch.
- `BP_GM_<ContentSetId>_Starter`, the game mode.
- `L_<ContentSetId>_Starter`, the map: a floor with collision, a built navigation mesh, minimap and
  vision bounds, fog of war, a player start and lighting.
- `DA_RTSGenerationManifest_<ContentSetId>`, the record of every asset Generate owns.

`ARTSHUD` creates the full in-game HUD for each player (resources, match clock, control groups,
selection panel, command card, notifications, minimap) without any project UI Blueprint.

Turn off **Match Setup > Generate Map** to generate only the unit, building and resource Blueprints.

The gameplay tags the plugin uses are registered by the plugin itself; you do not need to copy any
gameplay tag configuration into your project.

### Generate owns its output

Generate updates an asset only when its manifest proves Generate created it. If an asset exists
where Generate wants to write but the manifest does not list it, Generate stops and reports it
instead of overwriting it. Before changing anything, Generate checks every output path, compiles
every generated Blueprint without errors or warnings, checks that files are writable (asking source
control to check out read-only files), and only then saves. If saving fails part-way, it restores the
previous versions of the files it changed. See [Content Set Migration](CONTENT_SET_MIGRATION.md) for
the full list of conditions it reports.

Do not add graphs, components, variables or default values to generated assets. Put Blueprint logic
in a **Custom Blueprint**, or supply a complete class of your own through **Replace With Class**. See
[The Content Set](CONTENT_SET_GUIDE.md#add-blueprint-logic-to-a-unit-or-building).

## Version numbers

Every Content Set records a format version (**Advanced > Schema Version**). Generate has its own
version, recorded in the manifest. They change independently: a new generator version rebuilds
generated assets without touching your Content Set's data, while a new format version needs an
explicit upgrade of the Content Set. See [Content Set Migration](CONTENT_SET_MIGRATION.md).

## The plugin's sample map

To check the plugin works without installing anything, choose **RTS > Open the Plugin's Sample Map**
and press Play. It opens `/RealTimeStrategy/Starter/Complete/Generated/Maps/L_RTSComplete_Starter`,
the plugin's own copy of the starter game. The smaller sample is
`/RealTimeStrategy/Starter/Reference/Generated/Maps/L_RTSStarter_Starter`, with its Content Set at
`/RealTimeStrategy/Starter/Reference/DA_RTSStarter`. Turn on **Show Plugin Content** in the Content
Browser settings to see them. These are read-only plugin assets; edit your installed copy instead.

## Unreal's Manny

In a project made from the UE 5.8 Third Person template, or with its Characters content added, you
can give the starter units Unreal's Manny character. Manny stays your project's content; the plugin
finds it in `/Game` and does not include it.

1. Right-click your Content Set and choose **Use Unreal's Manny for Units**.
2. Read the result in the **RTS Authoring** page of the Message Log. The command checks for the
   exact UE 5.8 mesh, Animation Blueprint and clips before it changes anything. If they are missing,
   add the Third Person template content and run it again.
3. The command creates Anim Sets for the worker and the main combat unit under
   `<Content Set folder>/Presentation/Manny`, assigns Manny and its Animation Blueprint to those
   units, and marks the Content Set as changed.
4. Choose **Generate Game**, then **Play**.

You can undo the command, and run it again safely. For other characters use
**Use My Model for a Unit...**; see [Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md).

## HUD skin

The plugin uses its own HUD style, `DA_RTSHudStyle_Default`, without any project setting. To use
your own, create an **RTS HUD Style** asset and select it under
**Project Settings > Game > RTS HUD Style**, or set a style override on your own child of `ARTSHUD`.

Blueprint UI authors can derive from `ARTSHUD` and `URTSConsoleWidget`, replace the console or
minimap widget classes in Class Defaults, replace individual panels, and use the
**Console Widget Created** event for their own bindings. Turn off **Auto Create Console Widget**
only when your project supplies its own complete HUD. See [UI Data API](UI_DATA_API.md).

## Command line and C++ workflow

The same Content Set, validation and generation are available from the command line, for build
machines:

```text
UnrealEditor-Cmd Project.uproject -run=RTSValidateContentSet -ContentSet=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -ContentSet=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -ContentSet=/Game/MyRTS/DA_MyRules -ApplyUpgrade -OutputRoot=/Game/MyRTS/Generated
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MyRules
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MyRules -OutputRoot=/Game/MyRTS/Generated
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_NetRules -OutputRoot=/Game/MyRTS/Generated -MinHumanPlayers=2
UnrealEditor-Cmd Project.uproject -run=RTSGenerateContentSet -CreateStarter=/Game/MyRTS/DA_MannyRules -OutputRoot=/Game/MyRTS/Generated -Presentation=Manny -Observer
```

- `RTSValidateContentSet` accepts `-Fix` (apply the same fixes as **Fix Simple Problems**) and
  `-SaveFixes` (save them).
- `-CreateStarter` creates and saves a new starter Content Set, then generates it. Without
  `-OutputRoot`, the generated folders go into a `Generated` folder beside the new Content Set. It
  refuses to replace an existing asset. `-Force` resets an existing Content Set to the starter rules
  and regenerates it; assets of any other type are never overwritten.
- `-OutputRoot` moves only the generated folders of an existing Content Set; its Id stays the same.
- `-ApplyUpgrade` upgrades an older Content Set before generating; add `-UpgradeOnly` to upgrade
  without generating.
- `-Presentation=Manny` applies Unreal's Manny from the Third Person template, as described above.
- `-Observer` makes the generated map an AI-only match to watch. `-MinHumanPlayers=N` makes the
  match wait for N human players (at most the number of bases; must be 0 with `-Observer`).
  Connected humans take the first free bases, AI takes the rest, and later arrivals watch.

Every failure returns a non-zero exit code. Generation never changes your project's startup map; to
package a generated map as your game's first map, select it under
**Project Settings > Maps & Modes > Game Default Map**.

C++ code uses the same types: `URTSContentSet`, `FRTSUnitDefinition`, `FRTSBuildingDefinition`, and
the editor functions in `URTSContentSetEditorLibrary` (from an editor module). The runtime systems are
ordinary exported classes and components, so a C++ project can derive its own controllers, orders, AI,
components and widgets without using generated Blueprints.

### Complete classes of your own

A unit's or building's **Replace With Class** takes a complete class you made: a Character Blueprint
or `ACharacter` subclass for a unit, a Pawn Blueprint or `APawn` subclass for a building. Generate then
uses that exact class for training, construction, starting bases and the match settings, and no longer
creates `BP_<Id>` for that entry.

Your class is in charge of its components, mesh, weapons, what it gathers or builds, its costs and
what it trains. The Content Set entry keeps its Id, faction, role and match setup references.
**Check for Problems** checks that the class has every RTS component the entry's settings need and
reports each one that is missing. Builders and training buildings must already list the classes they
build or train; Generate never changes your class. Clear **Replace With Class** to go back to a
generated Blueprint.

## Testing tools

The RTS player controller uses `URTSCheatManager`, so in Play in Editor you can type
`SpawnInspect Worker` or `SpawnInspect Rifleman` in the console to spawn and select a unit for
inspection. `SpawnInspect` also accepts a full class path ending in `_C`. To add your own short
names, make a Blueprint child of `RTSCheatManager`, add them to **Inspectable Unit Classes**, and
select that class on your player controller. Spawned units belong to the first AI player if there is
one, otherwise to you. Unreal turns cheat managers off in Shipping builds.

## Swapping art

Gameplay size comes from the Content Set (a unit's **Size**, a building's **Footprint**), not from the
mesh. Replacing art never changes selection, pathing, placement, attack range, gathering distance or
navigation blocking.

- Static units: set **Static Mesh**.
- Animated units: set **Skeletal Mesh**, an **Anim Set** made for the same skeleton, and an
  **Animation Mode**: **Direct Anim Set (No Anim Blueprint)**, or **Custom Animation Blueprint** for
  your own Animation Blueprint driven by the unit's `RTSAnimComponent`.

The quickest way is **Use My Model for a Unit...** on the Content Set's right-click menu: pick the
unit and mesh, then **Create Anim Set from My Clips...** to map your clips. See
[the guided workflow](MULTIRIG_ANIM.md#guided-editor-workflow) for automatic animation triggers,
custom abilities, Animation Blueprints, sounds and effects.

Editor Utility Blueprints and C++ tools can call **Validate Imported Unit Presentation** before
changing a Content Set; it runs the same checks and returns coded findings. These checks confirm the
setup is valid, not that the animation looks right; always review the unit in play.

Manny, Mixamo, Meshy, marketplace and studio characters all use this same workflow once their clips
are imported for the chosen skeleton. The plugin never includes or depends on those assets.

## How the plugin's own content stays self-contained

The plugin's bundled assets never depend on your project's `/Game` content, so the plugin works in
a blank project. Your generated assets may use your own art from `/Game`; that is how reskinning
works.
