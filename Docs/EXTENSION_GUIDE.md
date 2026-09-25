# Extending with C++

This guide is for programmers who want to add game code in C++. You never need to edit the
plugin's source: every system below is extended from your own game module. Blueprint and C++ use
the same classes and rules, so you can mix them freely. For Blueprint-only work, start with the
[Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md).

## Set up your module

Add the runtime module to your game module's `.Build.cs`:

```csharp
PublicDependencyModuleNames.Add("RealTimeStrategy");
```

- Add `RealTimeStrategyExamples` only if you derive from or include the example classes.
- `RealTimeStrategyEditor` is an editor-only module. Depend on it only from an editor module of your
  own, for example to call Content Set functions from an editor tool.
- Include the header for each class you use, by its folder path, for example
  `#include "Combat/RTSHealthComponent.h"`. [C++ API Surface](CPP_API_SURFACE.md) lists the
  supported headers by area.

Keep your classes in your project's `Source` folder. Do not copy plugin classes into your project or
change plugin files: updates replace them.

## How the pieces fit

- **The Content Set is the source of truth for game data.** Units, buildings, resources, costs and
  tech requirements are data. **Generate Game** turns them into Blueprints (`BP_<Id>`), a map and a
  game mode. See [The Content Set](CONTENT_SET_GUIDE.md).
- **Units and buildings are ordinary actors with RTS components.** A unit is an `ACharacter`, a
  building an `APawn`. Health, ownership, attacks, gathering, construction and production are
  components such as `URTSHealthComponent`, `URTSOwnerComponent` and `URTSProductionComponent`.
- **The server decides.** Gameplay changes happen on the server. Clients send requests through
  their `ARTSPlayerController`, and results replicate back.

## Choose how to add your code

| Goal | C++ approach | Details |
| --- | --- | --- |
| React to deaths, training, construction or resources | Bind the component delegates (see below). | [Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md#events) lists the events. |
| Your own unit or building class | Derive from `ACharacter` or `APawn`, add the RTS components, and set it as the entry's **Replace With Class** in the Content Set. | [Starter Game and Generated Content](STARTER_RULESET.md#command-line-and-c-workflow) |
| A new resource or resource node | Derive from `URTSResourceType`, or build a node actor with `URTSResourceSourceComponent`, and set it as **Replace With Class** on the resource or resource source. | [Economy](ECONOMY.md) |
| A new command | Derive from `URTSOrder` and override `CanObeyOrder_Implementation`, `IsValidTarget_Implementation`, `IssueOrder_Implementation` and `GetDescription_Implementation`. | [Orders](ORDERS.md) |
| A new ability rule | Derive from `URTSAbilityComponent` and override its readiness, target or effect functions. | [Combat and Networking](COMBAT_NETWORKING.md) |
| AI strategy | Derive from `ARTSPlayerAIController`. Override `ExecuteStrategyThink_Implementation` to add decisions; also return false from `ShouldUseBuiltInStrategy_Implementation` to replace the built-in strategy entirely. | [AI and Match Policies](AI_MATCH_POLICIES.md) |
| Win conditions | Derive a game mode from `ARTSSkirmishGameMode` (or `ARTSGameMode`) and call `EliminatePlayer` or `CommitMatchResult` on the server. | [AI and Match Policies](AI_MATCH_POLICIES.md#c-win-conditions), [Match State](MATCH_STATE.md) |
| HUD panels | Derive from `ARTSHUD` or the bundled widgets, or bind the delegates from your own widgets. | [UI Data API](UI_DATA_API.md) |

Most classes offer small `BlueprintNativeEvent` hooks (functions ending in `_Implementation` in
C++). Override the hook, and call `Super::` to keep the plugin's behavior and add to it. Skip the
`Super::` call only where a guide says the hook is a complete replacement.

## React to gameplay events

Components expose Blueprint-assignable delegates. Bind them with `AddDynamic` to a `UFUNCTION`:

```cpp
// MyGameMode.h
UFUNCTION()
void HandleProductionFinished(AActor* Building, AActor* Product, int32 QueueIndex);

// MyGameMode.cpp
#include "Production/RTSProductionComponent.h"

void AMyGameMode::WatchBuilding(AActor* Building)
{
    if (URTSProductionComponent* Production = Building->FindComponentByClass<URTSProductionComponent>())
    {
        Production->OnProductionFinished.AddDynamic(this, &AMyGameMode::HandleProductionFinished);
    }
}
```

`URTSHealthComponent` also has a C++-only delegate, `OnKilledNative`, that you can bind without a
`UFUNCTION`. It fires just before the Blueprint **On Killed** event, on the server and again on
clients when the death replicates:

```cpp
#include "Combat/RTSHealthComponent.h"

void AMyGameMode::WatchUnit(AActor* Unit)
{
    if (URTSHealthComponent* Health = Unit->FindComponentByClass<URTSHealthComponent>())
    {
        Health->OnKilledNative.AddUObject(this, &AMyGameMode::HandleUnitKilled);
    }
}

void AMyGameMode::HandleUnitKilled(AActor* Unit, AController* PreviousOwner, AActor* DamageCauser)
{
    if (!HasAuthority())
    {
        return; // Clients receive the death too; only the server changes gameplay.
    }
    // Unit may be destroyed later this frame: do not keep the pointer.
}
```

Remove your bindings when your object is destroyed (in `EndPlay` or `Deinitialize`) if the
component can outlive it. `ARTSExampleObjectiveGameMode` in the examples module shows the complete
pattern: it binds `OnKilledNative` for each registered objective, unbinds in `EndPlay`, and
eliminates the objective's owner when it dies.

To find units as they appear, use the standard `UWorld::AddOnActorSpawnedHandler`, or bind
`OnProductionFinished` on the buildings that train them.

## Replace a generated unit with your own class

Set a unit's **Replace With Class** (in its **Blueprint** group in the Content Set) to your
`ACharacter` subclass, or a building's to your `APawn` subclass. Generate then stops creating
`BP_<Id>` for that entry and uses your class everywhere instead: training, construction and
starting bases.

Your class is then responsible for its own components and values: its mesh, stats, weapons, costs,
and what it gathers, builds or trains. The Content Set entry keeps only its Id, faction and role,
and the match setup's references to it. Your class needs every RTS component that the entry's
settings imply, for example `URTSGathererComponent` for a unit that gathers, or
`URTSProductionComponent` for a building that trains units, and a builder or production building
must already list the classes it builds or trains. **Check for Problems** lists every missing
component. Generate never changes your class. `ARTSExampleBuilderUnit` and `ARTSExampleFieldDepot` in the examples module are
complete, working examples of a hand-made unit and building.

If you only want to add logic to a generated unit, use a Blueprint **Custom Blueprint** instead.
It keeps all Content Set values. Generated unit Blueprints derive directly from `ACharacter` and
building Blueprints from `APawn`; there is no setting that gives them your own C++ parent class.

## Rules that keep your code safe in multiplayer

- **Change gameplay only on the server.** Check `HasAuthority()` before calling functions marked
  `BlueprintAuthorityOnly`, and never trust values a client sends: re-check ownership, cost, range,
  cooldown and visibility on the server right before you change anything.
- **Send player commands through the player controller.** Use the `ARTSPlayerController::Issue...`
  functions (for example `IssueMoveOrder` or `IssueProductionOrder`). For a custom order on the
  current selection, call `GetOrderSubmission()->IssueOrderToSelectedActors(Order)`. A `true` result
  means the request was sent, not that it succeeded.
- **Use the plugin's functions to change state.** Pay with `URTSPlayerResourcesComponent::PayResources`,
  change owners with `ARTSGameMode::TransferOwnership`, end the match with `CommitMatchResult`.
  Editing fields directly skips validation, replication and events.
- **Treat events as notifications.** Delegates report changes that already happened. Do not use
  them to authorize another change, and do not start the same transaction again from its own event.
- **Hold actors weakly.** Keep references to units and buildings in `TWeakObjectPtr` or `UPROPERTY`
  fields, and check them before use. Units die and are destroyed at any time.
- **Stay on the game thread.** Plugin objects are not thread-safe. Worker threads may process copied
  values only, then return to the game thread to apply the result.

[C++ Extension Rules](CPP_EXTENSION_CONTRACT.md) explains each rule with examples.
[Network Security](NETWORK_SECURITY.md) covers fog of war and what clients may see.

## Aircraft

The built-in **Move**, **Attack Move**, **Patrol** and **Attack** orders fly air units: characters
with a `URTSAirUnitComponent`, which switches them to the `MOVE_Flying` movement mode. Generate
adds this component to units that have **Is Air Unit** set in the Content Set. Generated units use their **Move Speed** as their flying speed; a class
set through **Replace With Class** must set its own movement speeds.

Aircraft keep their starting cruise height and route around obstacles by climbing over, crossing or
descending past them, with collision on. Nearby aircraft keep apart using their own 3D separation,
not Unreal's ground avoidance. An attack still needs a weapon that can hit the target's domain and
reach it in 3D. This is not general 3D navigation for enclosed levels: if no safe route exists, the
order fails.

Flight temporarily uses its own path follower when the controller has Unreal's standard
`UPathFollowingComponent`, and restores it afterwards. If your controller uses a custom path
follower, the built-in flight does not replace it and reports that it cannot fly the route; your
controller then needs its own flight movement. Custom order classes also need their own movement
code for aircraft. Test aircraft on your real maps and in a packaged multiplayer build.

## Before you ship an extension

1. Compile your project in Development and Shipping.
2. Choose **Check for Problems** on your Content Set and fix every error.
3. Play the generated map, then test the same behavior in a packaged build, with a second player
   if your game has multiplayer.

See [Blueprint and C++ Examples](EXAMPLES.md) for small working examples of every extension type,
and [API Change Policy](PUBLIC_API_CHANGE_POLICY.md) for how the C++ API may change between
versions.
