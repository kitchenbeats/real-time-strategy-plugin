# Real-Time Strategy Documentation

Welcome. This plugin gives you a complete, playable RTS in the style of StarCraft and Brood War, and
the tools to turn it into your own game in Blueprint, C++ or both. Follow the five steps below in
order; each builds on the one before.

## Learning path

### 1. Install and play the starter game

- [Getting Started](INSTALL_QUICKSTART.md) — install from Fab, enable the plugin, install the
  starter game, play a match, and make your first change.
- [What's Supported](SUPPORTED_PRODUCT_CONTRACT.md) — engine version, platforms, what is included
  and what support covers.

### 2. Make it your game with the Content Set

- [The Content Set](CONTENT_SET_GUIDE.md) — the one asset that defines your units, buildings,
  resources, factions and match setup; Generate Game, Play and Check for Problems.
- [Starter Game and Generated Content](STARTER_RULESET.md) — the starter game's map and rules, what
  Generate creates, and command-line generation.
- [Vendor-Neutral Skeletal Animation](MULTIRIG_ANIM.md) — bring your own animated characters.
- [Tech Tree](TECH_TREE.md) — requirements, upgrade levels and tiers, set up as data.
- [Starter Audio](STARTER_AUDIO.md) — replace the game's sounds.

### 3. Add Blueprint logic with Custom Blueprints

- [Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md) — where logic goes, and the events and
  nodes for deaths, training, selection, resources, orders and the match result.
- [UI Data API](UI_DATA_API.md) — build or restyle the HUD and bind it to game data.
- [Blueprint and C++ Examples](EXAMPLES.md) — small working examples of every kind of extension.
- [Blueprint API Rules](BLUEPRINT_API_CONTRACT.md) — how nodes report success, failure and
  server-only actions.

### 4. Extend with C++

- [Extending with C++](EXTENSION_GUIDE.md) — module setup, where your code goes, binding events and
  replacing units with your own classes.
- [C++ Extension Rules](CPP_EXTENSION_CONTRACT.md) — threading, server authority and object
  lifetime rules for your code.
- [C++ API Surface](CPP_API_SURFACE.md) — the supported headers, by area.

### 5. Ship it: packaging and multiplayer

- [Packaging and Multiplayer](PACKAGING.md) — package your game, set up Mac networking, and test
  local network and dedicated-server play.
- [Dedicated Server Lobbies](DEDICATED_LOBBIES.md) — run a server-managed lobby.
- [Troubleshooting](TROUBLESHOOTING.md) — fixes for the problems people hit most often.

## Reference

### Gameplay systems

- [Economy and Gathering](ECONOMY.md) — resources, gathering, carrying and drop-off, and how to customize them.
- [Construction and Production](CONSTRUCTION_PRODUCTION.md) — building placement, builders,
  training queues, costs, refunds and rally points.
- [Combat and Networking](COMBAT_NETWORKING.md) — health, attacks, abilities, stances and research,
  and how they replicate.
- [Orders](ORDERS.md) — how commands work, targeting, and writing your own orders.
- [Technology Progression](TECH_PROGRESSION.md) — research and upgrade state during a match.
- [Ownership and Teams](OWNERSHIP_TEAMS.md) — who owns each unit, teams and alliances.
- [Player Advantages](PLAYER_ADVANTAGES.md) — per-player handicaps and bonuses.
- [Gameplay Tags](GAMEPLAY_TAGS.md) — gameplay tags on units and buildings.
- [Vision](VISION.md) — sight, fog of war and remembered buildings.
- [Navigation and Flow Fields](NAVIGATION_FLOW_FIELDS.md) — unit movement and navigation options.
- [AI and Match Policies](AI_MATCH_POLICIES.md) — AI strategy, scouting, squads and win conditions.
- [Match State](MATCH_STATE.md) — match results and eliminations.

### Multiplayer

- [Network Security](NETWORK_SECURITY.md) — what each client can see, and what the server checks.
- [Chat](CHAT.md) — in-match chat and how to skin it.
- [Team Pings](TEAM_PINGS.md) — map pings between allies.

### Animation and diagnostics

- [Custom Animation Blueprint](ANIMBP_MANUAL_STEPS.md) — drive units with your own Animation
  Blueprint.
- [Locomotion Polish](AAA_LOCOMOTION_STEPS.md) — improve walking and running for close-up quality.
- [Diagnostics](DIAGNOSTICS_SPEC.md) — debug overlays, event logs and automated match checks.

### Versions and updates

- [Content Set Migration](CONTENT_SET_MIGRATION.md) — upgrade Content Sets after a plugin update.
- [API Change Policy](PUBLIC_API_CHANGE_POLICY.md) — how the API may change between versions.
- [API Change Log](API_CHANGE_LEDGER.md) — every API change, newest first, with what to do about it.
