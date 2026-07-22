# Extension Guide

This is the entry point for changing gameplay without editing plugin source. The bundled starter is
the working baseline; customer assets and subclasses live under the project's `/Game` and `Source`
folders. Blueprint-only and C++ projects use the same runtime classes, authority rules, and data.
`BLUEPRINT_API_CONTRACT.md` defines node categories, pins, request results, failure signals, and
authority behavior. `CPP_EXTENSION_CONTRACT.md` defines the corresponding native runtime rules.
`CPP_API_SURFACE.md` lists the stable entry headers and distinguishes additive extension seams from
complete subsystem replacement.

## Start with one working change

1. Play `/RealTimeStrategy/Starter/Reference/Generated/Maps/L_RTSStarter_Starter` once.
2. Create an **RTS Content Set**, validate it, and generate its project-owned starter map.
3. Change one Content Set definition, or create a customer-owned child of a generated class in a
   project folder outside the generated output root and select it from a customer-owned
   roster/map/configuration seam that keeps its generated parent expected.
4. Validate again and play the generated map before combining changes.

The reference Content Set and generated assets are organized under
`/RealTimeStrategy/Starter/Reference`. They are supported, inspectable examples—not hidden test
fixtures. `Docs/STARTER_RULESET.md` contains the complete generation and reskin workflow.

For one-feature-at-a-time learning, start in `/RealTimeStrategy/Examples/Blueprint`. Each asset is a
compiled Blueprint subclass of a matching class in `Source/RealTimeStrategyExamples`; duplicate it
into `/Game` before editing. `Docs/EXAMPLES.md` identifies the preserved authority and transaction
contract for every recipe.

## Capability map

| Goal | Blueprint path | C++ path | Detailed contract |
| --- | --- | --- | --- |
| Custom resource or gatherable source | Add a Resource and Resource Source to an RTS Content Set. When custom Blueprint behavior is needed, select a customer-owned child from a customer-owned roster/map seam, or supply a complete stable project-owned class through the applicable existing-class field. | Populate `FRTSResourceDefinition` / `FRTSResourceSourceDefinition`, or supply an existing `URTSResourceType` and source actor class. | `ECONOMY.md`, `STARTER_RULESET.md` |
| Custom unit | Add a Unit definition, assign greybox or skeletal/static presentation, then generate. For a hand-authored Character, set `ExistingUnitActorClass`. | Supply a complete `ACharacter` subclass through `ExistingUnitActorClass`, or construct the same exported gameplay components directly. | `STARTER_RULESET.md`, `MULTIRIG_ANIM.md` |
| Custom building | Add a Building definition and role; for a hand-authored Pawn, set `ExistingBuildingActorClass`. | Supply a complete `APawn` subclass or use the Content Set generator from the editor/commandlet API. | `STARTER_RULESET.md`, `CONSTRUCTION_PRODUCTION.md` |
| Custom order | Derive from `RTSOrder`; override availability, target validation, execution, and description; register it on the player controller/command card. | Derive from `URTSOrder` and override the matching `_Implementation` functions. | `ORDERS.md` |
| Custom ability | Add `RTSAbilityComponent`, author its catalog, then override readiness, target policy, or effect as required. Route player casts through the owning RTS player controller. | Derive from `URTSAbilityComponent` or call `ConfigureAbilities`; override the same native events. | `COMBAT_NETWORKING.md` |
| Custom AI policy | Derive from `RTSPlayerAIController`. Keep built-in strategy enabled for additive decisions, or disable it for a complete replacement; implement `Execute Strategy Think`. | Derive from `ARTSPlayerAIController` and override `ShouldUseBuiltInStrategy_Implementation` and/or `ExecuteStrategyThink_Implementation`. | `AI_MATCH_POLICIES.md` |
| Custom HUD panel | Derive the relevant native widget or bind component/subsystem delegates in a new User Widget; replace the panel class on the console/HUD. | Derive from the same exported widget, or bind the documented delegates directly. | `UI_DATA_API.md` |
| Custom win condition | On authority, call `Eliminate Player` for player-scoped failure or `Commit Match Result` for a winner/draw. | Call `ARTSGameMode::EliminatePlayer` or `CommitMatchResult` from the authoritative objective system. | `AI_MATCH_POLICIES.md`, `MATCH_STATE.md` |

## Shared runtime rules

`CPP_EXTENSION_CONTRACT.md` is the normative C++ contract for game-thread access, server authority,
UObject lifetime, delegate teardown, async work, replication callbacks, and override behavior. It
also applies when a native extension exposes a Blueprint implementation.

- Gameplay mutations happen on authority. Client UI checks improve feedback but never authorize a
  transaction.
- Actor ownership is the engine `Owner` relationship plus the replicated RTS owner/player-state
  contract. Use plugin transfer and controller request APIs instead of changing pointers directly.
- Component delegates and replicated state are the supported UI binding surface. Do not search the
  viewport or depend on a generated widget hierarchy.
- Soft references and project-owned subclasses keep the plugin independent of Manny, Mixamo,
  Meshy, marketplace, or studio assets.
- A Blueprint or C++ override must preserve the parent contract unless the documented seam says it
  is a complete replacement. Validation, payment, cooldown, range, team, and ownership checks remain
  required immediately before an authoritative mutation.

## Diagnosis before extension

Use **Validate Content Set**, the **RTS Authoring** Message Log, gameplay debugger categories, and
the match-check diagnostics before replacing a subsystem. A supported extension should be provable
in the generated starter map and then repeated in a packaged build; an editor-only success is not a
shipping result.

Generated bases and their manifest are inspectable build output, not customer customization
surfaces. Regeneration may replace their graphs, components, variables, and defaults. Keep every
customer-owned Blueprint outside the generated output root and route it through the Content Set's
existing-class fields or other documented composition points.
