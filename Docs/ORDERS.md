# Orders

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

The controller's named high-level order helpers remain the supported convenience facades. For a
generic `FRTSOrderData` affecting the current selection, call **Get Order Submission** on the
controller and then **Issue Order to Selected Actors**. The C++ equivalent is
`Controller->GetOrderSubmission()->IssueOrderToSelectedActors(Order)`. The controller creates one
replicated native `URTSOrderSubmissionComponent`; do not add or create another instance.

The submission component owns the generic selection-to-request path, structural validation,
bounded authority admission, and packet-safe batching/RPC transport. `ARTSPlayerController` still
owns named high-level commands, `IsOrderClassAllowedFromClient`, targeting flow, and the public
order/error event surface. A `true` submission result means local admission succeeded and the
request was sent; it does not report how many pawns authority accepted or guarantee execution or
eventual completion.

The bundled command card automatically calls `BeginOrderTargeting` for actor- and location-targeted
orders. The controller changes to the crosshair cursor, uses left click to confirm and right click to
cancel, and preserves an invalid request so the player can choose again. `ConfirmTargetingAt` is the
same Blueprint-callable completion path for a custom cursor, gamepad selector, or minimap UI.
`OnTargetingStarted` and `OnTargetingEnded` let a replacement HUD reskin the prompt without replacing
the command logic.

## Inspect accepted command plans

For a custom HUD, obtain the owning controller's **Get Order Submission** component, bind
**On Accepted Order Plan Changed**, and read **Get Accepted Order Plan** in that event. The event
has no parameters; also read the getter once after binding and unbind when the widget is removed.
Inspection follows selection automatically and does not issue or change gameplay orders.

`FRTSAcceptedOrderPlan` describes exactly one selected, owned, living pawn controlled by
`ARTSPawnAIController`. Check `FocusedPawn` before displaying it. An empty result means no usable
acknowledged snapshot: selection may be unsupported, changed, or still awaiting the server.
Clear previous text and markers immediately; an empty snapshot does not prove an idle unit or
successful submission. For a valid snapshot, a null `CurrentOrder.OrderClass` represents idle/Stop.
Selection changes, multiple selection, death, ownership loss and observer mode invalidate the view.
Inspection requires matching controller ownership and `RTSOwnerComponent` player-state ownership.
For gameplay ownership transfers, call the game mode's **Transfer Ownership** API; it updates both
identities together. Changing only the owner component invalidates inspection until they agree.

`CurrentOrder` and `QueuedOrders` contain `FRTSOrderPlanStep` values: `OrderClass`, the order's
`Index` payload, and `bHasDestination`/`Destination`. Preserve array order when numbering pending
commands; `Index` is not their queue position. `QueuedCount` and `QueueCapacity` report the actual
pending count and configured capacity (16 by default). Transport includes at most 64 pending
steps; `bTruncated` means only the display snapshot was shortened. The native presentation bounds
its list and world cues to 12 commands and reports additional pending commands. These bounds do
not change queue capacity or execution.

Snapshots replicate only to the owning player. Actor-target steps contain no target actor
reference or identifier. Their destination is available only for a friendly target or one currently
visible to that player; otherwise `bHasDestination` is false. Draw a destination only when that flag
is true and break route lines across unavailable steps. Lines describe command sequence, not a
navigation path or guaranteed reachability. Authority refreshes are bounded to five per second;
`Revision` and `InspectionId` identify snapshot changes and focus acknowledgements, not completion
of individual orders. Budget-delayed inspection retries are bounded; a timeout reports an error
and selecting the unit again retries inspection.

When a submitted queued order exceeds capacity, the existing controller error channel reports
the number of full unit queues. Rejected queues retain their accepted commands; eligible units in a
mixed selection can still accept the order. The public submission Boolean retains its admission
semantics described above. Use the accepted snapshot to present server state and
`OnErrorOccurredEvent` for rejection feedback.

## C++ order workflow

Derive from `URTSOrder`, configure defaults in the constructor, and override the native
implementation functions:

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
