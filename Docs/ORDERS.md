# Orders: Blueprint and C++ Extension Contract

`URTSOrder` is the reusable command definition for units and buildings. The built-in move, attack,
gather, return-resource, construction, boarding, stop, and rally-point orders use the same public contract
available to customer projects.

`URTSLoadIntoContainerOrder` is included in the controller's default right-click priority. A unit
with `URTSContainableComponent` can target a friendly `URTSContainerComponent`; its AI approaches
the container, revalidates ownership/capacity on authority, and boards when it reaches the authored
range. No project Behavior Tree edits are required. Override the normal order extension points or
the container's `CanLoadActor` rule for cargo classes, transport permissions, or faction mechanics.

## Blueprint order workflow

1. Create a Blueprint class derived from `RTSOrder`.
2. Set its target type, group execution, gameplay-tag requirements, display text, and icons in Class
   Defaults.
3. Override **Can Obey Order** for unit-specific availability rules.
4. Override **Is Valid Target** for context-sensitive actor or location validation.
5. Override **Issue Order** to perform the authoritative command. Call the parent implementation
   when the order should enter the standard `RTSPawnAIController` order queue.
6. Optionally override **Get Description** for selection- or index-dependent UI text.
7. Add the class to the RTS Player Controller's `DefaultOrders` and the command-card widget's
   `CandidateOrderClasses`, or supply it through a custom controller/widget subclass.

`RTSOrderTargetData` and `RTSOrderTagRequirements` are Blueprint-readable and writable. The
`RTSOrderLibrary` supplies Blueprint nodes for availability checks, target-data construction,
presentation data, and authority-only order execution.

An order submitted by a network client is accepted only if the pawn belongs to that controller and
the controller's `Is Order Class Allowed From Client` policy permits the order class. The default
policy permits `DefaultOrders` plus the built-in construction and stop commands. Add custom client
orders to `DefaultOrders`, or override that policy deliberately.

The bundled command card automatically calls `BeginOrderTargeting` for actor- and location-targeted
orders. The controller changes to the crosshair cursor, uses left click to confirm and right click to
cancel, and preserves an invalid request so the player can choose again. `ConfirmTargetingAt` is the
same Blueprint-callable completion path for a custom cursor, gamepad selector, or minimap UI.
`OnTargetingStarted` and `OnTargetingEnded` let a replacement HUD reskin the prompt without replacing
the command logic.

## C++ order workflow

Derive from `URTSOrder`, configure defaults in the constructor, and override the native
implementation seams:

```cpp
virtual bool CanObeyOrder_Implementation(const AActor* OrderedActor, int32 Index) const override;
virtual bool IsValidTarget_Implementation(
    const AActor* OrderedActor,
    const FRTSOrderTargetData& TargetData,
    int32 Index) const override;
virtual void IssueOrder_Implementation(
    AActor* OrderedActor,
    const FRTSOrderTargetData& TargetData,
    int32 Index) const override;
virtual FText GetDescription_Implementation(const AActor* OrderedActor, int32 Index) const override;
```

These are `BlueprintNativeEvent` functions, so the normal `CanObeyOrder`, `IsValidTarget`,
`IssueOrder`, and `GetDescription` calls dispatch correctly to either a Blueprint override or the
C++ implementation.

## Authority and validation

`IssueOrder` runs on the server for the plugin's normal multiplayer input path. Custom order
implementations must treat all actor references, indices, resources, ranges, cooldowns, technology
requirements, and target state as untrusted and revalidate them immediately before changing game
state. UI availability and client-side target checks are feedback, not security boundaries.

Use `ARTSPlayerController::IsOrderClassAllowedFromClient_Implementation` to extend the inbound
class allow policy in C++, or override **Is Order Class Allowed From Client** in a controller
Blueprint. Call the parent policy when extending the built-in allow-list.
