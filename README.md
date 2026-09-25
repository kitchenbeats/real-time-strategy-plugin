# Real-Time Strategy Plugin for Unreal Engine 5.8

A framework for making real-time strategy games in Unreal Engine 5.8, in the style of StarCraft and
Brood War, together with a complete starter game you install into your project and make your own.
It works in Blueprint-only and C++ projects and needs no other content.

This repository holds the plugin's documentation and its support tracker. The same guides ship
inside the plugin under `Docs/`, and **RTS > Guide** in the editor opens them.

## Start here

1. Enable **Real-Time Strategy** in **Edit > Plugins** and restart the editor.
2. Choose **RTS > Install the Starter Game**. This copies a complete game into
   `/Game/RTSStarterGame`, builds it and opens its map. Press **Play**, then **Start match**.
3. Choose **RTS > Customize My Game** to open your game's **Content Set**. Change units, buildings,
   costs and art there, then choose **Generate Game** and **Play**.

The full walkthrough is in [Getting Started](Docs/INSTALL_QUICKSTART.md), and
[the documentation index](Docs/README.md) lists every guide in the order you will need them:

1. Install and play the starter game
2. Make it your game with the Content Set
3. Add Blueprint logic with Custom Blueprints
4. Extend with C++
5. Ship it: packaging and multiplayer

## Support

Open an issue on this repository's [issue tracker](../../issues). Please include your engine
version, platform, plugin version (see `RealTimeStrategy.uplugin`), and the relevant log excerpt.
[Troubleshooting](Docs/TROUBLESHOOTING.md) covers the most common first-hour problems.

## Licensing

The plugin is a commercial product distributed under the store's standard license terms. This
repository contains documentation only; it does not grant any license to the plugin itself. The
plugin incorporates MIT-licensed work; see `THIRD_PARTY_NOTICES.md` inside the plugin for
attributions.

Copyright © 2026 Jeremy Hanlon.
