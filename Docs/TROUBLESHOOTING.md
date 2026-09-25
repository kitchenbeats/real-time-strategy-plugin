# Troubleshooting

Problems people run into in their first hours with the plugin, and how to fix them. When something
fails, look first at the **Output Log** (**Window > Output Log**; plugin messages use the `LogRTS`
category) and the **Message Log** (**Window > Message Log**, page **RTS Authoring**).

## The plugin does not appear or does not load

1. **Edit > Plugins**: search for **Real-Time Strategy**, make sure it is ticked, and restart the
   editor.
2. The plugin must match your engine version. Messages such as "missing modules" or "built with a
   different engine version" mean the installed plugin does not match UE 5.8. Install the 5.8
   version again from the Fab Library.
3. If you copied the plugin into your project, `RealTimeStrategy.uplugin` must sit directly in
   `<YourProject>/Plugins/RealTimeStrategy`. An extra folder level (for example
   `Plugins/RealTimeStrategy-1.3.0/RealTimeStrategy`) stops Unreal from finding it.
4. Keep exactly one copy of the plugin. Remove a project copy if it is also installed in the engine,
   and never mix files from two versions.
5. In a C++ project, a failed build stops the plugin from loading. Build the project in your IDE and
   fix the first compiler or linker error in the list; later errors are often caused by the first.

## The RTS menu or the Content Set commands are missing

- The **RTS** menu and the **Tools > Real-Time Strategy** entries come from the plugin's editor
  module. They appear only after the plugin is enabled and the editor restarted.
- **Generate Game**, **Check for Problems** and the other **RTS Game** commands appear when you
  right-click a **Content Set** asset, not other assets. They are also buttons at the top of the
  Content Set editor.
- A packaged game never contains these editor tools. That is expected.

## I cannot find the plugin's sample content

Plugin content is hidden in the Content Browser by default. Open the Content Browser **Settings**
menu and turn on **Show Plugin Content**. The sample game is at
`/RealTimeStrategy/Starter/Complete/Generated/Maps/L_RTSComplete_Starter`, or open it with
**RTS > Open the Plugin's Sample Map**. The plugin's samples are read-only; install the starter game
to get a copy you can edit.

## The map looks empty in the editor

This is normal. Bases, workers, buildings and resource nodes are created when the match starts.
In the editor viewport the map holds only the ground, lights and a few volumes (navigation, vision,
minimap and fog of war). Press **Play** to see the match.

## Play says there is no generated map yet

A Content Set has no map until you choose **Generate Game**. If the message stays after generating,
open **Match Setup** in the Content Set and make sure **Generate Map** is on, then generate again.

## My changes do not show up in the game

- Choose **Generate Game** after changing the Content Set. Play uses the generated assets, not the
  Content Set directly.
- Play the generated map. The Content Set's **Play** button opens the right one. If you use the
  editor's own Play button, check that the open level is your `L_<Id>_Starter` map.

## My edits to a generated Blueprint disappeared

Everything in the `Generated` folders is rebuilt by **Generate Game**, which replaces changes made
directly to those assets. Opening a generated unit or building Blueprint shows a notification saying
so, and **Generate Game** asks **Replace edited Blueprints?** before replacing edits it detects.

Put Blueprint logic in a **Custom Blueprint** instead: open the unit or building in the Content Set
and choose **Create Custom Blueprint** (in its **Blueprint** group), or use the button on that
notification. Custom Blueprints are saved in a `Custom` folder next to the Content Set, and
Generate never overwrites them. See
[The Content Set](CONTENT_SET_GUIDE.md#add-blueprint-logic-to-a-unit-or-building).

Change numbers, names, costs and art in the Content Set itself, not in any Blueprint.

## Create Custom Blueprint shows an error

- "'X' has not been generated yet": choose **Generate Game** first, then create the Custom Blueprint.
- "... is inside a folder that Generate owns": the Content Set itself is saved inside its own
  generated output folder. Move the Content Set to a folder of your own, then try again.
- "An asset named ... already exists and is not a child of ...": an unrelated asset already uses
  the name `BP_<Id>_Custom` in the `Custom` folder. Rename or delete it, then try again.

## Generate Game or Check for Problems reports errors

The **RTS Authoring** page of the Message Log lists each problem with its cause and the fix. Fix
the first error, choose **Check for Problems** again, and generate once it passes. Common ones:

- **An Id that does not exist.** A **Trains Units**, **Can Build**, **Required Buildings** or match
  setup field names an Id that no entry has, usually after an Id was renamed. Pick the right entry
  from the drop-down list.
- **An asset Generate did not create sits in a generated folder.** Generate refuses to overwrite
  assets it does not own. Move your own assets out of the `Generated` folders.
- **Custom Blueprint problems.** The Custom Blueprint was deleted or moved, is not a child of the
  generated `BP_<Id>`, sits inside a generated folder, or the entry also has **Replace With Class**
  set. Fix the field as the message says; **Create Custom Blueprint** makes a correct one.
- **Replace With Class is missing components.** A class set through **Replace With Class** must
  contain every RTS component the entry needs. The message names the missing component.
- **Files are read-only.** With source control, check out the generated files (or let Unreal check
  them out) before you generate.

**Fix Simple Problems** (right-click the Content Set) repairs values that have only one sensible
answer and then checks again. It never invents design values such as health, costs or Ids.

## Units do not move, or walk through walls

Units move on Unreal's navigation mesh. Generated maps come with it built. In a map you made
yourself, or after you move the floor or add obstacles:

1. Make sure the ground has collision. The navigation mesh is built only on surfaces that have
   collision.
2. Place a **Nav Mesh Bounds Volume** that covers the whole playable area.
3. In **World Settings**, under **Navigation System Config**, set **Navigation System Class** to
   `RTSNavigationSystem`. Generated maps do this for you. It lets buildings placed during a match
   cut holes in the navigation mesh.
4. Choose **Build > Build Paths**, then press **P** in the viewport to show the navigation mesh.
   Walkable ground turns green.

While playing, open the console (backtick key) and run `rts.diag.audit`. It lists setup problems
in the Output Log, each line starting with `[audit]` and giving the cause and the fix:

- `NAV.NO_DATA`: the map has no navigation data. Add or enlarge a **Nav Mesh Bounds Volume** and
  rebuild paths.
- `NAV.PAWN_OFF_MESH`: a unit stands outside the navigation mesh. Check where it spawns, its size
  and the floor's collision.
- `NAV.PAWN_FALLING`: a unit is falling near the navigation mesh. Check its spawn height and the
  floor's collision.

`rts.diag.audit.fix` also applies safe runtime fixes, such as giving an AI controller to a unit
that has none. It never edits your map. `rts.diag.draw 1` draws order lines and marks stuck units
in the world.

## Workers do not gather or do not bring resources back

- The worker's **Gathers** list (Content Set, **Economy** group) must include the resource.
- A building must accept the resource: its **Accepts Resources** list must include it. The main
  base usually accepts every resource.
- A resource source with **Requires Extractor** (like a gas geyser) needs a building with
  **Extracts Resource** built on top before anyone can gather.
- If workers stop at the edge of a large building without dropping off, raise that building's
  **Reach Margin** (in its **Footprint**).

## The AI attacks too early, or kills my workers

On the setup screen, choose **Passive** or **Easy** under **AI difficulty**, or **None (sandbox)**
under **AI opponents**. Change the preselected difficulty with **Match Setup > Default AI Difficulty**
in the Content Set. To make workers defend themselves, turn on **Worker Fights Back** on the worker
unit (Content Set, **Combat** group).

## My character model looks wrong

- **Grey or default material in a packaged game.** Open the character's materials and, under
  **Usage**, turn on **Used with Skeletal Mesh**, then save them.
- **Facing the wrong way, too high or too low.** Adjust the unit's **Visual Rotation** and
  **Visual Offset** in the Content Set. The art is scaled to the unit's **Size**; change **Size**, not
  the mesh scale.
- **The model is rejected when you apply it.** The skeletal mesh, its Anim Set and every clip must
  use the same skeleton.
- **Direct Anim Set (No Anim Blueprint)** needs saved, full-pose, in-place clips. It rejects
  additive and root-motion clips, because the RTS movement code moves the unit. Use
  **Custom Animation Blueprint** for layered animation, aim offsets, IK or motion matching.
- **Use Unreal's Manny for Units cannot find Manny.** The command needs the UE 5.8 Third Person
  template's Characters content in your project. Add it (**+ Add > Add Feature or Content Pack**,
  then the Third Person template content), save, and run the command again.

See [Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md) for the full checklist.

## It works in the editor but not in the packaged game

- Check **Project Settings > Maps & Modes > Game Default Map**: it must be your generated map.
- Make sure every asset your game loads is cooked. Assets referenced only by name (for example from
  your own data tables of paths) must be added to the packaging settings.
- Search the packaged game's log for the first "missing package", Blueprint error or `LogRTS` error.

See [Packaging and Multiplayer](PACKAGING.md).

## After updating or disabling the plugin

Do not save assets that depend on the plugin (your Content Set, generated Blueprints, maps) while
the plugin is disabled or failing to load. Turn it back on, restart, open your Content Set, choose
**Check for Problems**, then **Generate Game**. If Generate says the Content Set needs an upgrade,
follow [Content Set Migration](CONTENT_SET_MIGRATION.md).

## Asking for help

Use **RTS > Report a Problem** to open the support page. Include:

- the plugin version, your UE 5.8 version, operating system and target platform;
- whether your project is Blueprint-only or C++;
- the steps that reproduce the problem, starting from the installed starter game or a new Content
  Set if possible;
- the first error from the Output Log or Message Log, with the lines around it; and
- for character problems, where the model came from, its import settings, skeleton and animation
  mode.

Remove passwords, account names and other private details from logs before you share them.
[What's Supported](SUPPORTED_PRODUCT_CONTRACT.md) describes what support covers.
