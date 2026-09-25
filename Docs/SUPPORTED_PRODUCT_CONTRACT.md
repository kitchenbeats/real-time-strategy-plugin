# What's Supported

This page lists what the Real-Time Strategy plugin supports, what it includes, and what support
covers. Check it before you promise a feature or platform in your own game.

## Engine, projects and platforms

| | Supported |
| --- | --- |
| Engine | Unreal Engine 5.8. Other engine versions are not supported by this release. |
| Project types | Blueprint-only and C++ projects. |
| Editor and target platforms | Windows (Win64) and macOS with Apple Silicon. The plugin is tested on macOS with Apple Silicon. Epic compiles it for Windows during Fab review, but it has not yet been play-tested on Windows; please report any Windows problem on the support tracker. Intel Macs, Linux, consoles, mobile and XR are not supported. |
| Rendering | The engine's standard desktop renderer. |
| Input | Keyboard and mouse. Gamepad, touch, split-screen and VR are not included. |

## What the plugin includes

- **A complete starter game** you install into your project and customize: one faction, eight units,
  eight buildings, two resources, construction, training, research, supply, aircraft, siege units,
  healing, defense towers, AI opponents, a setup screen, a game menu with settings, victory and
  defeat. Its art is simple placeholder shapes built from engine meshes and plugin materials.
- **The RTS framework**: selection, orders and shift-queued commands, gathering, construction,
  production, research, abilities, combat, fog of war, minimap, HUD, AI players and match rules, as
  Blueprint-accessible components and classes.
- **Editor tools**: the Content Set editor, **Generate Game**, **Check for Problems**, Custom
  Blueprints, and tools to apply your own models and animations.
- **Small Blueprint and C++ examples** of each way to extend the game.
- **Editable UI source art** in `SourceArt/UIKit`: icon, panel and font sources for reskinning the
  HUD, with their licenses.

## Characters and art

The plugin ships only placeholder art. You can use:

- **Static meshes** for any unit, building or resource node.
- **Rigged characters** with the plugin's animation workflow, either with no Animation Blueprint
  (**Direct Anim Set**) or with your own **Custom Animation Blueprint**.
- **Unreal's Manny**, taken from the Third Person template content in your own project. The plugin
  does not include or redistribute Manny.
- **Characters from other sources** (Mixamo, Meshy, marketplaces, your own studio) once you have
  imported them with a working skeleton and clips. You are responsible for the licenses of the art
  you use. A character that works in one project is not a guarantee that every character from the
  same source works; check each one with **Check for Problems** and in play.

## Multiplayer

- Local network play from the included menus: one player hosts, others join, everyone marks ready.
- Dedicated servers with the included lobby. See [Dedicated Server Lobbies](DEDICATED_LOBBIES.md).
- The server decides all gameplay. Fog of war and private information stay on the server through
  the plugin's Replication Graph setup. If your project turns on Unreal's Iris replication, you must
  provide an equivalent visibility filter yourself or keep Iris off.
- Not included: online matchmaking, accounts, backend services, anti-cheat, campaigns and saved
  games. You can add them with your own systems or third-party services.

Test every multiplayer setup you plan to ship in a packaged build. See
[Packaging and Multiplayer](PACKAGING.md).

## Navigation and scale

- Units use Unreal's standard navigation mesh. Generated maps set it up for you.
- The optional flow-field movement modes (**Military Only** and **All Pawns**) are experimental. The
  default, **Stock Navigation**, is the supported mode. See
  [Navigation and Flow Fields](NAVIGATION_FLOW_FIELDS.md).
- Very large or streamed worlds (World Partition) are not supported yet.
- The plugin does not promise a unit count or frame rate. Measure your own game in a packaged build
  on your target hardware.

## What support covers

Support covers the plugin's runtime, editor tools, included content and the documented Blueprint
and C++ extension points, on the supported engine version and platforms.

Support does not cover:

- modified engine source or modified plugin source;
- plugin internals that the documentation does not describe;
- fixing or retargeting third-party art and animations;
- your own backend services, networking middleware or custom Iris filters; and
- project code that changes gameplay on clients or bypasses the documented server rules.

Your game's balance, factions, art, licenses, localization and platform certification are yours.

To report a problem, use **RTS > Report a Problem** in the editor and include the details listed in
[Troubleshooting](TROUBLESHOOTING.md#asking-for-help).

## Updates and compatibility

Content Sets carry a version number. When a plugin update changes their format, the plugin
provides an upgrade step that keeps your data, and Generate refuses to run until it is applied; see
[Content Set Migration](CONTENT_SET_MIGRATION.md). How
the Blueprint and C++ API may change between versions is described in the
[API Change Policy](PUBLIC_API_CHANGE_POLICY.md), and every change is listed in the
[API Change Log](API_CHANGE_LEDGER.md).
