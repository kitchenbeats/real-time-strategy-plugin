# Small Blueprint and C++ Examples

The examples are supported, focused extension recipes. They are deliberately separate from the
complete starter so a customer can learn one policy without reverse-engineering a whole match.
Every Blueprint asset under `/RealTimeStrategy/Examples/Blueprint` is compiled from a matching
Blueprintable class in `Source/RealTimeStrategyExamples`.

Duplicate a Blueprint example into the project's `/Game` folder before changing it. C++ projects
can derive from the matching exported class or copy the small policy into a project-owned class.
Do not edit the plugin copy: upgrades replace plugin files.

| Recipe | Blueprint asset | C++ class | What it demonstrates |
| --- | --- | --- | --- |
| Wounded-target focus fire | `Orders/BP_ExampleFocusFireOrder` | `URTSExampleFocusFireOrder` | Adds a health threshold while preserving the attack order's tag, visibility, relationship, invulnerability, ownership, and authoritative execution checks. |
| Focus-fire order registration | `Orders/BP_ExampleOrderPlayerController` | `ARTSExampleOrderPlayerController` | Adds the focus-fire order to the normal default-order routing without replacing the core order catalog. |
| Healthy-veteran ability gate | `Combat/BP_ExampleVeteranAbilityComponent` | `URTSExampleVeteranAbilityComponent` | Adds caster-health readiness to the normal ability validation, payment, cooldown, targeting, and server transaction. |
| Emergency reinforcement AI | `AI/BP_ExampleReinforcementAIController` | `ARTSExampleReinforcementAIController` | Adds a rate-limited production policy to the built-in strategy and uses the normal affordability, technology, and production APIs. |
| Critical-objective defeat | `Match/BP_ExampleObjectiveGameMode` | `ARTSExampleObjectiveGameMode` | Registers health-bearing objectives on authority and eliminates the associated controller only after the health component commits its killed transaction. |
| Supplemental HUD panel | `UI/BP_ExampleHUD` | `ARTSExampleHUD` | Adds one local-player widget after the native console is ready and removes it during teardown without replacing the core HUD. |
| Crystal resource type | `Economy/BP_ExampleCrystalResource` | `URTSExampleCrystalResource` | Defines code-owned localizable presentation for a resource type without project assets. |
| Crystal resource node | `Economy/BP_ExampleCrystalNode` | `ARTSExampleCrystalNode` | Composes a visible, selectable, replicated gather source whose primitive visual can be replaced independently. |
| Builder unit | `Units/BP_ExampleBuilderUnit` | `ARTSExampleBuilderUnit` | Composes the complete unit contract, gathers the example resource, returns cargo, and constructs the example depot. |
| Field depot building | `Buildings/BP_ExampleFieldDepot` | `ARTSExampleFieldDepot` | Composes the complete constructible-building contract, authoritative footprint/nav blocker, and matching resource drain. |

## Use the Blueprint recipes

1. Enable **Show Plugin Content** and open `RealTimeStrategy/Examples/Blueprint`.
2. Duplicate the desired asset into `/Game/RTS/Examples` (or the project's own feature folder).
3. Change the exposed example settings in Class Defaults.
4. Register the order or component through the normal unit/controller catalog, select the AI or
   GameMode class in the skirmish configuration, or select the HUD class in the GameMode defaults.
5. Validate the Content Set, play the generated map, and repeat the behavior in a packaged build.

The Blueprint classes are executable assets, not screenshots: automation loads every asset, checks
its intended native parent and generated class, and rejects compiler warnings. The release gate then
repeats those tests from an isolated installed plugin.

## C++ contract

Include the relevant public header and add `RealTimeStrategyExamples` to the project's module
dependencies. The example module is runtime-safe and compiled for Editor, Development, and Shipping.
The core framework remains in `RealTimeStrategy`; production code does not depend on the example
module. Override the smallest policy method possible and call the parent implementation whenever
the recipe is additive. Gameplay mutation remains server-authoritative. Every native recipe also
follows the game-thread, lifetime, delegate, async, and transaction rules in
`CPP_EXTENSION_CONTRACT.md`.

## Register the composition example

The resource, node, worker, and depot are intentionally wired to one another in native class
defaults. In an RTS Content Set, add:

- resource id `ExampleCrystal` with **Existing Resource Class** set to
  the native `RTSExampleCrystalResource` class;
- source id `ExampleCrystalNode`, resource id `ExampleCrystal`, and **Existing Source Actor Class**
  set to `BP_ExampleCrystalNode`;
- unit id `ExampleBuilder`, **Existing Unit Actor Class** set to `BP_ExampleBuilderUnit`, one gather
  entry for `ExampleCrystal`, **Can Build** enabled, and constructible id `ExampleFieldDepot`;
- building id `ExampleFieldDepot`, **Existing Building Actor Class** set to
  `BP_ExampleFieldDepot`, and accepted resource id `ExampleCrystal`.

The Content Set fields own stable ids and scenario membership; the existing classes remain
authoritative for components, costs, gather/build catalogs, footprint, visual, and presentation.
Validation fails if the declared capabilities and class composition diverge. Duplicate the node,
worker, and depot Blueprints into `/Game` before reskinning them, then point the Content Set at the
project copies.
Resource classes are transaction identities, not interchangeable presentation skins. If a project
instead derives its own resource class from `BP_ExampleCrystalResource`, it must select that exact
class consistently in the node source, worker gather policy, depot drain, and every related cost.

The supported examples now cover all advertised extension categories: resource/source composition,
unit composition, building composition, orders, abilities, strategic AI, objective conditions, and
HUD injection. Broader rig/vendor workflow validation and manual Blueprint usability review remain
separate production-readiness gates.
