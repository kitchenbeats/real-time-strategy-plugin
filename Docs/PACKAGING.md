# Packaging and Multiplayer

This guide covers turning your RTS into a packaged game and testing it, including multiplayer.
Package only after your game plays correctly in the editor. Even then, always test the packaged
game itself: Play in Editor does not show missing cooked assets, the wrong startup map or signing
problems.

## Before you package

1. Save everything: Content Sets, Anim Sets, Custom Blueprints, maps and settings.
2. Open your Content Set, choose **Check for Problems** and fix every error in the
   **RTS Authoring** page of the Message Log (**Window > Message Log**).
3. Choose **Generate Game** after your last change to the Content Set, so the generated assets
   match it.
4. Choose **Play** and play a full match: select units, move, gather, build, train, fight, win or
   lose, and restart.
5. Open **Project Settings > Maps & Modes** and set **Game Default Map** to your generated map
   (for the installed starter game, `/Game/RTSStarterGame/Generated/Maps/L_RTSComplete_Starter`).
   **Install the Starter Game** fills this in only when it was empty or still on an engine
   template map. Generating again never changes it.
6. Make sure every asset your game loads gets cooked. Assets the game finds only by name or path at
   run time must be added under **Project Settings > Packaging** (for example in
   **Additional Asset Directories to Cook**).
7. In a C++ project, check that your module lists `RealTimeStrategy` in its `.Build.cs`
   dependencies (and `RealTimeStrategyExamples` only if you use the examples).
8. Package into an empty output folder, start the packaged game directly, and keep its log.

## Package the game

Use Unreal's normal **Platforms** menu, pick your platform, then **Package Project**. A
Blueprint-only project does not need a C++ module to use the plugin. If packaging reports that
plugin modules are missing or were built for another engine version, reinstall the plugin version
that matches your engine; do not convert the project to C++ to work around it.

In a C++ project, build with the compiler and SDK that UE 5.8 requires for your platform. Keep
editor-only code (for example tools that call Content Set functions) in an editor module; your game
module depends only on `RealTimeStrategy`. Package both Development and Shipping, and test the
Shipping build, which is what players run.

## Mac Shipping networking

On a Mac, a sandboxed Shipping game needs permission to host and join network games.
**Install the Starter Game** sets this up: it copies the engine's server entitlement template into
your project's `Build/Mac/Resources` folder and selects it for Shipping builds. If your project
already had its own signing settings or entitlement files, the plugin keeps them and tells you to
review them.

If you use your own entitlements, a game that hosts and joins needs both
`com.apple.security.network.client` and `com.apple.security.network.server`. Keep any other
entitlements your project needs, and keep the entitlement file in source control.

Package again into a fresh folder after changing signing settings; an existing app keeps its old
signature. To check the packaged app's entitlements, run:

```bash
codesign --display --entitlements - --xml "/path/to/YourGame.app"
```

Then test hosting and joining with the Shipping build. A Development build does not use the
Shipping signing settings.

## Art and animation checks

- Move or rename your own assets only through the Content Browser, and fix up redirectors
  (right-click the folder, **Fix Up Redirectors**) before the final package.
- **Direct Anim Set** clips must be saved, full-pose and in-place, and made for the unit's skeleton.
  In the packaged game, watch idle, walking, working, attacking, getting hit and dying.
- A **Custom Animation Blueprint** must compile for the unit's skeleton. Check poses, foot contact,
  materials, LODs and how the unit reads from the RTS camera distance.
- Materials used on skeletal meshes need **Used with Skeletal Mesh** turned on, or characters show
  the default material in the packaged game.
- Art from Manny, Mixamo, Meshy, marketplaces or your studio stays under its own license. The
  plugin does not give you rights to redistribute it.

## Multiplayer

The included game supports network play without extra setup:

- **Local network.** On the setup screen, press **Esc** (**Multiplayer & settings**). One player
  chooses **Host LAN game**; the others enter the host's address and port and choose
  **Join LAN game**. Each player picks a base or chooses to spectate, marks themselves **Ready**,
  and the host starts the match. Players who join a running match watch as spectators.
- **Dedicated server.** See [Dedicated Server Lobbies](DEDICATED_LOBBIES.md).
- **Minimum players.** **Match Setup > Minimum Human Players to Start** in the Content Set makes
  the match wait for that many human players.

Test every kind of multiplayer game you plan to ship (hosting from a player's machine, a dedicated
server, or both) in packaged builds; one working in Play in Editor does not prove the other works.

The server decides all gameplay. Keep your own gameplay changes on the server and send player
commands through the player controller, as described in [Extending with C++](EXTENSION_GUIDE.md) and
the [Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md). The plugin uses Unreal's
Replication Graph so that each player receives only what their units can see. If you enable Unreal's
Iris replication, you must provide an equivalent visibility filter or turn Iris off.

Matchmaking, accounts, backend services, anti-cheat, campaigns and saved games are not included;
add them with your own systems. Player settings (controls, audio, display) are saved locally by the
included menus; see [UI Data API](UI_DATA_API.md).

## Platforms

This release is tested on macOS with Apple Silicon and Unreal Engine 5.8. See
[What's Supported](SUPPORTED_PRODUCT_CONTRACT.md) for the current platform list before you
promise a platform to your players.

## Check the packaged game

On a clean machine or user account, check that the packaged game:

- starts on your map without needing the editor or your project folder;
- shows the HUD and creates the human and AI players you expect;
- plays a full match and can restart;
- shows your art and animations in every state;
- has no fatal errors, missing packages, Blueprint errors or repeated warnings in its log; and
- runs well enough on your target hardware with the number of units you plan to allow.

If something fails, see [Troubleshooting](TROUBLESHOOTING.md#it-works-in-the-editor-but-not-in-the-packaged-game).
