# The Content Set

A **Content Set** is one data asset that describes your whole game: its units, buildings,
resources, factions, match setup and shared rules. You edit the Content Set, then choose
**Generate Game**. The plugin builds the playable game from it: a Blueprint for every unit,
building and resource, plus a map, a game mode and the match settings.

You never have to write code or edit generated Blueprints to change numbers, names, costs, tech
requirements or art. Change the Content Set and generate again.

## Open your Content Set

- After **Install the Starter Game**, choose **RTS > Customize My Game** (also in
  **Tools > Real-Time Strategy**). This opens `/Game/RTSStarterGame/DA_RTSStarter`.
- Any other Content Set: double-click it in the Content Browser.
- To make a new one: **+ Add > Gameplay**, then **RTS Content Set** in the **RTS** section. It
  starts with a small working game (see [Getting Started](INSTALL_QUICKSTART.md#5-start-a-new-game-from-scratch-optional)).

The plugin's own copies under `/RealTimeStrategy/Starter` are read-only samples. Edit the copy in
your project instead.

## The Actions row

The top of the Content Set editor has four buttons:

| Button | What it does |
| --- | --- |
| **Generate Game** | Checks the Content Set for problems, then rebuilds every generated asset from it. |
| **Play** | Opens this game's generated map and starts Play in Editor. |
| **Check for Problems** | Looks for mistakes without generating, and explains how to fix each one. |
| **Guide** | Opens this documentation. |

The usual loop is: change something, **Generate Game**, **Play**.

The same commands appear when you right-click a Content Set in the Content Browser, in the
**RTS Game** section at the top of the menu:

| Menu entry | What it does |
| --- | --- |
| **Generate Game** | Same as the button. Works on several selected Content Sets at once. |
| **Check for Problems** | Same as the button. |
| **Fix Simple Problems** | Fixes problems that have only one sensible answer, such as a negative cost or an empty output folder, then checks again. It never invents design values such as health, costs or ids. You can undo it with **Edit > Undo**. |
| **Use My Model for a Unit...** | Picks a mesh and animations from your project for one unit. See [Use your own models](#use-your-own-models). |
| **Use Unreal's Manny for Units** | Gives the worker and the main combat unit Unreal's Manny character and animations. Needs the Third Person template's Characters content in your project. |

Results appear in a notification. Details, including every problem found, go to the
**RTS Authoring** page of the Message Log (**Window > Message Log**). The Message Log opens by
itself when **Check for Problems** or **Generate Game** finds errors.

## Sections of the Content Set

| Section | What it holds |
| --- | --- |
| **Actions** | The buttons above. |
| **Game** | **Display Name** (your game's name as players see it) and **Id** (a short internal name used in generated asset names, such as `L_<Id>_Starter`). Changing the Id makes Generate create a new set of assets. |
| **Units** | Every unit in the game. |
| **Buildings** | Every building in the game. |
| **Resources** | **Resources** (what players collect and spend, such as minerals and gas) and **Resource Sources** (the nodes placed on the map, such as mineral fields and gas geysers). |
| **Match Setup** | The generated map and match: factions, starting bases, number of players, AI difficulty and map layout. |
| **Gameplay Defaults** | Values shared by the whole game: sight, attack and chase distances, gathering, production queues, refunds, selection priority and health bar size. |
| **Presentation** | **Placeholder Art When Missing** and the health bar, progress bar, selection ring and selection sound used by every unit and building. |
| **Advanced** | Output folders, the AI controller used by generated units, the asset's format version, and **Reset to Starter Game...**. |

Hover over any field to see what it does. Distances are in centimeters and times in seconds.

### Ids and display names

Every unit, building, resource, resource source, faction and research line has two names:

- **Display Name** is what players see. Change it freely.
- **Id** is the short internal name other entries use to refer to it: a building's
  **Trains Units** list, a unit's **Required Buildings**, the starting worker in **Match Setup**,
  and so on. Keep the Id when you only want a new name for players.

Fields that refer to another entry are drop-down lists of the Ids defined in this Content Set,
labelled with their display names. If you rename an Id, pick the new Id again in every field that
used the old one; **Check for Problems** lists any reference that no longer matches.

## Units

Each unit entry is titled with its display name, with its Id shown next to it. Open an entry to see
its fields. The identity fields show first; the rest are grouped:

| Group | Main fields |
| --- | --- |
| **Identity** | **Id**, **Faction**, **Role**, **Display Name**, **Description**. Role describes the unit's job; the AI sends units with the Role `Worker` to gather. |
| **Appearance** | **Visual Mode** (your own mesh, or a generated placeholder shape), **Skeletal Mesh** or **Static Mesh**, **Portrait**, **Size** (collision radius and height; the art is scaled to fit), **Visual Rotation** and **Visual Offset** (to fix imported meshes that face the wrong way or sit too high), and how long the body stays after death. |
| **Animation** | **Anim Set** (the unit's clips), **Animation Mode** (**Direct Anim Set (No Anim Blueprint)** or **Custom Animation Blueprint**) and the **Animation Blueprint Class** for the custom mode. |
| **Stats** | **Maximum Health**, **Armor**, **Unit Size**, **Move Speed**, **Is Air Unit**, **Interaction Reach** (how close the unit gets to gather, build or repair) and starting gameplay tags. |
| **Combat** | **Attacks** (one entry per weapon: damage, range, cooldown, projectile, splash, and whether it hits ground, air or both), the **Weapon**, **Armor** and **Shield Upgrade** research lines it benefits from, shields, **Stances** (for example a siege mode), **Starting Stance**, and **Worker Fights Back**. |
| **Economy** | **Gathers** (which resources it collects, how much per trip, how fast), **Can Repair** and the repair rate and cost. |
| **Production** | Training cost, whether it is paid up front or over time, **Production Time**, **Supply Cost** and **Required Buildings**. |
| **Construction** | **Can Build** and the list of buildings it can construct. |
| **Abilities** | An energy pool and spells such as healing or area damage. Each ability gets a command card button. |
| **Blueprint** | **Custom Blueprint** and **Replace With Class**. See [Add Blueprint logic](#add-blueprint-logic-to-a-unit-or-building). |

**Worker Fights Back** controls what a gathering unit does when it or a nearby ally is attacked.
Off (the StarCraft behavior) keeps workers mining until you order them to fight. On makes them
fight back and then return to mining once the attacker is dead or gone. Soldiers always fight back.

### Add a unit

1. In **Units**, choose the **+** button to add an entry. Give it an **Id** such as `Scout` and a
   **Display Name**.
2. Set its **Faction** to the same faction as the building that will train it, then its **Stats**
   and **Attacks**.
3. Leave **Visual Mode** on **Use Greybox Primitive** to get a placeholder shape, or assign a mesh.
4. Set its training **Production** cost and time.
5. Open the building that should train it and add the unit to that building's **Trains Units**.
6. Choose **Generate Game**, then **Play**.

A unit that no building trains, and that is not the starting worker, never appears in a match.
**Check for Problems** reports an error when a **Trains Units**, **Can Build** or
**Required Buildings** entry names an Id that does not exist.

## Buildings

Building entries work the same way. The groups that differ from units:

| Group | Main fields |
| --- | --- |
| **Identity** | **Role**: what the building does for the player and the AI. **Townhall** is the main base, **Supply** raises the supply cap, **Production** trains units, **Tech** unlocks things, **Extractor** sits on a resource node, **Resource Depot** accepts returned resources, **Tower** defends. |
| **Appearance** | **Static Mesh**, **Portrait** and **Footprint**: the size on the ground used for collision, placement and pathing. **Reach Margin** lets workers gather, drop off, build or repair from a little further away, which helps with large buildings such as a town hall. |
| **Stats** | **Maximum Health**, **Armor**, **Supply Provided**. |
| **Combat** | **Attacks** for a defensive building. A building with attacks shoots enemies in range by itself. |
| **Construction** | Cost, **Construction Time**, grid size and **Required Buildings**. |
| **Production** | **Trains Units** and **Research Options** (upgrades such as weapon and armor levels, with cost, time, number of levels and requirements). |
| **Economy** | **Accepts Resources** (makes it a drop-off point), **Extracts Resource** (makes it an extractor, like a refinery on a gas geyser) and extractor settings. |

A worker can build a building when the worker has **Can Build** turned on and either lists the
building under **Can Build** or leaves that list empty (which allows every building of its faction).

### Tech requirements

Use **Required Buildings** on units, buildings and research options to build a tech tree. For
example, give the Factory the Barracks as a required building, and it stays locked on the command
card until the player owns a finished Barracks. Research options can also require other research
levels and a specific building for a specific level.

To make a unit stronger with research, add a research option to a building (its **Id** is the
research line, for example `InfantryWeapons`), then pick that line as the unit's
**Weapon Upgrade**, **Armor Upgrade** or **Shield Upgrade**.

## Resources and resource sources

- **Resources** define what players collect and spend: **Id**, **Display Name**, HUD **Icon** and
  **Color**, and **Starting Amount**.
- **Resource Sources** define the nodes on the map: which resource they give, their mesh and
  **Size**, **Maximum Resources**, how many workers can gather at once, and whether a building
  must be built on top first (**Requires Extractor**, like a gas geyser).

Costs everywhere else refer to resources by Id.

## Match Setup: factions, bases and the map

**Match Setup** controls the map and match that Generate creates. Its groups:

- **Map.** **Generate Map** must be on for Generate to create a map, game mode and match
  settings. **Num Players** (bases on the map, human and AI together), **Maximum Participants**,
  **Playable Half Extent** (half the map width in cm; 6000 makes a 120 m square map) and
  **Arena Preset** (**Open Arena**, a flat map with bases in a circle, or **Twin Crossings**, a
  two-player map with two routes around a central obstacle. Twin Crossings needs 2 players, a
  **Maximum Participants** of 2 and a **Playable Half Extent** of 6000 cm).
- **Players.** **Factions** (see below), **Require Player Setup** (show the setup screen before
  the match; needs at least one faction), **Default AI Difficulty** (preselected on the setup
  screen: **Passive**, **Easy** or **Normal**; the default is **Easy**), **Human Controls First Base**
  (turn off to watch AI players fight each other) and **Minimum Human Players to Start** for
  multiplayer.
- **Starting Bases.** **Starting Building**, **Starting Worker**, **Starting Workers Per Base**, the
  **Resource Node** placed at every base, and an optional **Second Resource Node** (such as a gas
  geyser) with how many to place per base and per expansion.
- **Advanced.** **Initial Simulation Speed**. Keep it at 1 for normal play.

### Factions

Each faction has an **Id**, **Display Name**, **Description** and **Crest** shown on the setup
screen, plus its own **Starting Building** and **Starting Worker**. Units and buildings join a
faction through their **Faction** field. Workers build only buildings of their own faction, and
buildings train only units of their own faction. Leave **Factions** empty for a single-faction game
that uses the starting building and worker from **Starting Bases**.

## What Generate creates

Generate writes into the folders set under **Advanced** (**Generated Content Root** and
**Output Paths**). For the installed starter game that is `/Game/RTSStarterGame/Generated`:

| Folder | Assets |
| --- | --- |
| `Units` | `BP_<Id>` for every unit, such as `BP_Worker`. |
| `Buildings` | `BP_<Id>` for every building, such as `BP_Barracks`. |
| `Resources` | `BP_Res_<Id>` for every resource and `BP_Src_<Id>` for every resource source. |
| `Maps` | `L_<ContentSetId>_Starter` (the map), `BP_GM_<ContentSetId>_Starter` (its game mode) and `DA_Skirmish_<ContentSetId>` (its match settings). |
| (root) | `DA_RTSGenerationManifest_<ContentSetId>`, the record of which assets Generate owns. |

Everything in these folders belongs to Generate. **Generate rebuilds these assets from the
Content Set and replaces any change you make to them directly.** When you open a generated
Blueprint, a notification says so and offers **Create Custom Blueprint** or
**Open Custom Blueprint**, **Open Content Set** and **Dismiss**. If you did edit a generated unit or
building Blueprint, Generate lists it and asks **Replace edited Blueprints?** before going ahead.

Do not save your own assets inside the generated folders. Generate stops with an error when it
finds an asset there that it did not create.

## Add Blueprint logic to a unit or building

To give a unit or building its own graphs, variables or components, use a **Custom Blueprint**:

1. Generate the game at least once.
2. Open the unit or building in the Content Set, expand its **Blueprint** group and choose
   **Create Custom Blueprint** (the row is labelled **Your Blueprint**). You can also choose it
   from the notification that appears when you open the generated Blueprint.
3. The plugin creates `<Content Set folder>/Custom/BP_<Id>_Custom`, a child of the generated
   `BP_<Id>`, sets it as the entry's **Custom Blueprint**, saves both, and opens it. For the
   installed starter game the worker's is `/Game/RTSStarterGame/Custom/BP_Worker_Custom`.
4. Its Event Graph starts with example events wired to a **Print String**: **On Killed** for every
   unit and building, and **On Production Finished** for buildings that train units. Replace them
   with your logic or delete them.
5. Generate never overwrites your Custom Blueprint, and players get it everywhere the unit or
   building appears: training, construction and starting bases. Every value in the Content Set
   still applies to it, because it inherits from the generated Blueprint.

Once a Custom Blueprint exists, the button reads **Open Custom Blueprint**.
[Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md) shows how to react to death, training,
selection and other events there.

**Replace With Class** is for advanced cases: it replaces the unit or building with a class you
built entirely yourself, in Blueprint or C++. Generate then stops creating `BP_<Id>` for that
entry, and the entry's stats no longer apply. Your class must contain the RTS components the
entry's abilities need; **Check for Problems** lists any that are missing. Set only one of
**Custom Blueprint** and **Replace With Class**. See [Extending with C++](EXTENSION_GUIDE.md).

## Use your own models

Gameplay size comes from the Content Set (a unit's **Size**, a building's **Footprint**), not from
the mesh. Swapping art never changes pathing, selection, placement or ranges.

- **Static meshes.** Set the entry's **Visual Mode** to **Use Assigned Art** and pick a
  **Static Mesh**. This works for units, buildings and resource sources.
- **Animated characters.** Right-click the Content Set and choose **Use My Model for a Unit...**.
  Pick the unit and a **Skeletal Mesh**, then either pick an existing **RTS Anim Set** or choose
  **Create Anim Set from My Clips...** to map your Idle, Walk and optional Run, attack, gather,
  build, death and hit clips. Choose **Direct Anim Set (No Anim Blueprint)** for the simplest setup,
  or **Custom Animation Blueprint** to use your own Animation Blueprint. Choose
  **Apply Presentation**, then **Generate Game**.
- **Unreal's Manny.** In a project made from the Third Person template (or with its Characters
  content added), right-click the Content Set and choose **Use Unreal's Manny for Units**.

Details, clip requirements and animation events are in
[Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md).

## Other assets you can create

The **+ Add > Gameplay** menu lists these in its **RTS** section:

| Asset | Use it for |
| --- | --- |
| **RTS Content Set** | A game or ruleset. |
| **RTS Anim Set** | Walk, run, attack and other clips for one skeleton. Assign it to a unit together with that unit's mesh. |
| **RTS HUD Style** | Colors, fonts, sizes and sounds for the in-game HUD. Select it in **Project Settings > Game > RTS HUD Style**. See [UI Data API](UI_DATA_API.md). |
| **RTS Command Card Layout** | Which button goes in which slot of the command card, and its hotkey. |
| **RTS Skirmish Definition** | Match rules for a map you made by hand: bases, starting workers, factions and AI. Generated maps get one automatically. |

## Start over

**Advanced > Reset to Starter Game...** replaces every unit, building, resource and setting in this
Content Set with the starter game's. It asks first. It keeps the Content Set's name, Id, output
folders and Custom Blueprints. You can undo it with **Edit > Undo** until you close the editor; once
you save and close, it cannot be undone. It is not available on the plugin's read-only sample.

## Automate from the command line

Build machines can check and generate a Content Set without opening the editor:

```text
UnrealEditor-Cmd MyProject.uproject -run=RTSValidateContentSet -ContentSet=/Game/MyRTS/DA_MyRTS
UnrealEditor-Cmd MyProject.uproject -run=RTSGenerateContentSet -ContentSet=/Game/MyRTS/DA_MyRTS
```

Add `-Fix` to `RTSValidateContentSet` to apply the same fixes as **Fix Simple Problems**, and
`-SaveFixes` to save them. Both commands return a non-zero exit code on failure. More options are
listed in [Starter Game and Generated Content](STARTER_RULESET.md#command-line-and-c-workflow).
Editor Utility Blueprints can call the same functions from the **RTS|Authoring** node category.
