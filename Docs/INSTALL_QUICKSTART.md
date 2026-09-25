# Getting Started

This guide takes you from installing the plugin to playing your own copy of the included RTS game.
It takes about ten minutes. You need Unreal Engine 5.8.

## 1. Install the plugin

### From Fab

1. Open the Epic Games Launcher and go to your **Fab Library** (under **Unreal Engine > Library**).
2. Find **Real-Time Strategy** and choose **Install to Engine**. Pick your 5.8 engine.
   The Launcher installs the plugin into the engine's `Engine/Plugins/Marketplace` folder, so every
   project that uses that engine can enable it.

### From a plugin folder

If you received the plugin as a folder instead, close the editor and copy the whole
`RealTimeStrategy` folder to `<YourProject>/Plugins/RealTimeStrategy`. The file
`RealTimeStrategy.uplugin` must sit directly inside that folder, with no extra folder level in
between. Keep only one copy of the plugin: never have it both in the engine and in the project.

## 2. Enable it

1. Open your project in Unreal Engine 5.8. Any template works, including **Blank**.
2. Choose **Edit > Plugins**, search for **Real-Time Strategy**, tick it, and restart the editor.

The plugin also turns on the engine plugins it needs (Enhanced Input, Niagara and Replication
Graph). You do not need to enable them yourself.

After the restart, a notification offers three choices:

- **Install the Starter Game** copies a complete, playable RTS into your project. Start here.
- **Try It First** opens the plugin's own sample map so you can press Play straight away.
- **Later** closes the notification.

The same actions stay available in two places: the **RTS** menu in the editor's main menu bar,
and the **Real-Time Strategy** section of the **Tools** menu.

| Menu entry | What it does |
| --- | --- |
| **Install the Starter Game** | Copies the starter game into `/Game/RTSStarterGame`, builds it and opens its map. |
| **Customize My Game** | Opens your installed game's Content Set, where you change units, buildings, costs and art. |
| **Open the Plugin's Sample Map** | Opens the read-only sample game inside the plugin. Useful to check the plugin works. |
| **Rename Game** | Sets your game's name for its menus and the packaged game's window title. |
| **Guide** | Opens this documentation. |
| **Online Documentation** and **Report a Problem** | Open the plugin's web pages in your browser. |

## 3. Install and play the starter game

1. Choose **Install the Starter Game** (in the notification, or **RTS > Install the Starter Game**).
   If you have unsaved work, the editor asks you to save it first.
2. The plugin copies the starter game into `/Game/RTSStarterGame`, generates its units, buildings
   and map, and opens the map `/Game/RTSStarterGame/Generated/Maps/L_RTSComplete_Starter`. A
   notification says **Starter game installed — it is your game now**.
3. If the notification asks you to restart Unreal Editor (it does this the first time, to switch
   on the RTS mouse cursor), restart and reopen the map.
4. Press **Play**. The **Skirmish** setup screen appears:
   - **Your name** is the name other players see.
   - **Your faction** is fixed while the game has one faction.
   - **AI opponents** sets how many computer players join. **None (sandbox)** gives you an empty
     map to try things out.
   - **AI difficulty** is **Passive** (builds up but never attacks), **Easy** or **Normal**. It is
     available when there is at least one AI opponent.
5. Choose **Start match**.

The copy in `/Game/RTSStarterGame` belongs to your project. Change anything in it. Running
**Install the Starter Game** again keeps your changes: it reuses your existing Content Set and
rebuilds the game from it.

The editor viewport shows only the ground, lights and a few volumes. Bases, workers and resources
are created when the match starts, so press Play to see them.

### Playing

- Drag with the left mouse button to select units. Click a unit or building to select it.
- Right-click the ground to move, an enemy to attack, or a mineral field to gather.
- Select a worker to see what it can build on the command card. Select a building to train units or
  research upgrades. Every command card button shows its hotkey.
- **Esc** opens the game menu: **Resume**, **Settings**, **Surrender**, **Return to setup** and
  **Quit game**. **Settings** changes sound, display, edge scrolling, camera speed and key bindings.
- The **? Controls** button on the HUD lists the main hotkeys.

The starter game has one faction with eight units and eight buildings, two resources (minerals and
gas), construction, training, research, supply, aircraft, siege units, healing and defense towers.
Its art is simple placeholder shapes so you can replace it with your own.

### Multiplayer on a local network

On the setup screen, press **Esc** (**Multiplayer & settings**). Under **Multiplayer**, one player
chooses **Host LAN game**. The others type the host's address and port and choose **Join LAN game**.
Each player picks a base and marks themselves **Ready**, then the host starts the match. For a
server without a local player, see [Dedicated Server Lobbies](DEDICATED_LOBBIES.md).

## 4. Make it your game

Choose **RTS > Customize My Game**. This opens your game's **Content Set**, the one asset that
describes your whole game: its units, buildings, resources, factions and match setup.

1. Expand **Units**, open **Worker**, and change its **Display Name** to something new, such as
   "Drone".
2. Choose **Generate Game** at the top of the Content Set. The plugin rebuilds your game from the
   Content Set in a second or two.
3. Choose **Play**. The Content Set opens its map and starts the game. The worker's train button,
   tooltip and selection panel now show its new name.

[The Content Set](CONTENT_SET_GUIDE.md) explains every section, how to add units and buildings,
and how to swap in your own models.

To give a unit Blueprint logic of its own, open it in the Content Set and choose
**Create Custom Blueprint**. See [Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md).

## 5. Start a new game from scratch (optional)

You do not have to start from the starter game. To begin with a small ruleset:

1. In the Content Browser, choose **+ Add > Gameplay**, then **RTS Content Set** in the **RTS**
   section. Save it in a folder of your own, for example `/Game/MyRTS`.
2. Open it and choose **Generate Game**, then **Play**.

A new Content Set starts with a small working game: minerals, a mineral field, a worker, a
rifleman, a town hall, a barracks and a supply depot, all with placeholder shapes. Its **Id** is the
asset name without the `DA_` prefix, and Generate writes into `/Game/RTS/Generated/<Id>`, with the
map at `/Game/RTS/Generated/<Id>/Maps/L_<Id>_Starter`. This small game has no factions, so the
match starts straight away without the setup screen. Add a faction and turn on
**Require Player Setup** under **Match Setup** to get the setup screen.

## 6. C++ projects

Enabling the plugin is enough for Blueprint-only projects. In a C++ project, add the runtime module
to your game module's `.Build.cs` before you include plugin headers:

```csharp
PublicDependencyModuleNames.Add("RealTimeStrategy");
```

Add `RealTimeStrategyExamples` too only if your code uses the example classes. The editor module
is not a runtime dependency. See [Extending with C++](EXTENSION_GUIDE.md).

## 7. Before you package

**Install the Starter Game** sets your installed map as the **Game Default Map** and **Editor
Startup Map**, but only where those settings were still empty or on the engine's template maps.
If your project already had its own startup map, choose your game's map under
**Project Settings > Maps & Modes**. Then follow [Packaging and Multiplayer](PACKAGING.md), and
always test the packaged game, not only Play in Editor.

## Update, disable or remove the plugin

Back up or commit your project (including `Config` and `Content`) before you change the plugin.

- **Update.** Close the editor and update the plugin (from the Fab Library, or by replacing the
  whole plugin folder; never mix files from two versions). Reopen the project, open your Content
  Set, choose **Check for Problems**, then **Generate Game**. If Generate reports that the Content
  Set needs an upgrade, follow [Content Set Migration](CONTENT_SET_MIGRATION.md).
- **Disable.** Assets made from plugin classes (your Content Set, generated Blueprints, maps) cannot
  load while the plugin is off. Do not save, redirect or delete them while it is disabled. Turn the
  plugin back on to use them again.
- **Remove.** Removing the plugin does not convert or delete your project's assets. Keep a backup
  so you can reinstall the same version if you need those assets later.

## Next steps

- [Documentation index](README.md) lists every guide.
- [What's Supported](SUPPORTED_PRODUCT_CONTRACT.md) lists engine versions, platforms and what
  support covers.
- [Troubleshooting](TROUBLESHOOTING.md) covers the problems people hit in their first hour.
