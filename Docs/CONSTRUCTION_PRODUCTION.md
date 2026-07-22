# Construction and Production Extension Contract

The default construction and production components provide server-authoritative building
placement, builder assignment, immediate or over-time costs, parallel production queues, supply
and technology gates, cancellation refunds, spawned-product ownership, and rally points.

## Blueprint extension points

`RTSBuilderComponent`, `RTSConstructionSiteComponent`, and `RTSProductionComponent` are
Blueprintable. Their core policies and state transitions are Blueprint Native Events:

- Builder assignment, building creation, and leaving a site
- Site admission, construction start, completion, and cancellation
- Product admission, queue selection, production-time calculation, queueing, completion, and
  cancellation

State-changing events are authority-only. Call the parent implementation to retain the standard
queue, cost, replication, diagnostics, delegate, and spawn behavior around project-specific rules.
Pure policy events are safe for command-card and placement feedback on clients, but the server
re-evaluates them before changing state.

Construction start, finish, and cancel return whether their authoritative transaction committed and
reject callback re-entry. State changes wake dormant sites and replicate with
`On Construction State Changed`; assigned builder identities replicate only to the owning connection
and drive `On Assigned Builders Changed`. Completion commits finished state, payment finalization,
workforce release/consumption, tags, and replication before customer callbacks. Cancellation commits
the paid-fraction and workforce snapshot before external calls, completes all wallet credits before
construction refund/cancel delegates, then destroys the site as an economic action—not a combat
kill—so it cannot award kill credit or trigger elimination logic.

`CanConstructBuildingAt` is the shared construction-admission policy. The begin-construction order
uses it before accepting a destination, and the builder calls it again immediately before spawning
the site so latency, moving units, or simultaneous builders cannot make an old preview decision
authoritative. Its native implementation validates the class allowlist and coordinates, then checks
shape or authored `RTSFootprintComponent` bounds against live dynamic blockers. Blueprint and C++
overrides should call the parent implementation before adding faction territory, creep, power,
adjacency, resource-node, or other game-specific placement rules.

Accepted construction sites spawn with `AlwaysSpawn` at the already validated transform, rather
than using the generic unit policy that may adjust around collision. The builder verifies the final
actor transform before broadcasting success and destroys a site that moved itself during Blueprint
construction. `ARTSGameMode::SpawnActorForPlayer` exposes collision handling to C++ callers and sets
the actor owner in spawn parameters, so startup logic sees the owning controller from its first
lifecycle callback. The helper rejects missing or invalid controllers; player construction and
production cannot silently degrade into neutral actors.

Completed production is transactional. The default `CanCompleteProduction` implementation holds a
paid unit at 100% while its player is supply blocked or temporarily lacks valid ownership, and it is
a Blueprint/C++ override point for project-specific completion gates. Queue state is committed before
customer completion callbacks, preventing reentrant listeners from completing or canceling the same
unit twice. If spawning ultimately fails, immediate and pay-over-time costs are refunded and
`OnProductionFailed` is broadcast. `PayCustom` costs are never fabricated by the plugin; custom
economies use that failure event to perform their own exact recovery.

Native projects do not need reflection or Blueprint helper subclasses to author the same data.
`ConfigureProduction` sets a validated product catalog and queue layout without discarding active
orders. `ConfigureProductionCost` and `ConfigureSupply` are constructor-safe transactional APIs;
invalid values leave the previous class data intact. Supply profiles can be authored from C++
default subobjects, generated content, Blueprint defaults, or authority setup before component
registration and replicate with the owning actor. They become immutable once registered so clients,
AI, production, and UI cannot disagree about population accounting. Blueprint authors keep the corresponding
organized component Details panels and can also reconfigure an idle production catalog at authority
runtime.

Supply census uses saturating 64-bit accumulation before returning Blueprint-friendly integers,
clamps provided supply to the ruleset's hard cap of 200, rejects mismatched world/controller
contexts, and uses subtraction-based production admission. Even deliberately extreme component
values therefore remain supply-blocked instead of wrapping signed arithmetic into a false success.

The placement cursor has matching native/runtime configuration entry points:
`ConfigureGridMetrics`, `ConfigurePlacementChecks`, and `ConfigureRangeIndicator`. They validate the
same bounds shown in the Blueprint Details panel and refuse unsafe changes while a grid or range
preview is active, so code-only cursor subclasses can be authored without private-property access.

Builders and construction sites provide the same parity. `ConfigureBuilder` authors the normalized
building catalog and staging behavior. `ConfigureConstructionEconomy`,
`ConfigureConstructionWorkforce`, and `ConfigureConstructionPresentation` cover cost/time/refunds,
builder contribution and consumption, starting health/autostart, footprint/range preview, and finish
audio. All reject invalid or active-state mutation without partially changing the component.

Configured constructible building and available product classes accept Blueprint and C++ derived
classes. This lets a project specialize or reskin a supported unit/building class without copying
every builder or producer definition. A deliberate Blueprint policy override can implement faction
restrictions, add-on requirements, alternate queue selection, accelerated production, cooperative
construction, or other game-specific rules.

Rally-point actor, location, and clear operations are exposed as authority-only Blueprint nodes.
Construction-site attack-range preview is exposed as a pure Blueprint query.

## Placement preview contract

`ARTSPlayerController::BeginBuildingPlacement` validates the building and cursor classes, observer
state, world, and available resources, and returns whether a fully configured preview was created.
Confirmation is transactional: the preview remains active until a selected builder accepts the
construction order. Cancellation and controller teardown clear the entire placement state.
The native cursor includes its own transform root, so a C++ project does not need a Blueprint-added
scene component merely to obtain functional cursor movement.

The bundled building cursor snaps before validating, and the controller uses that exact snapped
location for feedback, order dispatch, and placement delegates. Moving to a new cell immediately
refreshes collision/navigation checks; a configurable stationary refresh detects moving blockers
without repeating every grid trace every rendered frame. `MaximumGridWidthAndHeight` defaults to
32 cells per axis to prevent an accidental Blueprint footprint from creating unbounded synchronous
queries. Cursor subclasses may deliberately raise that limit (up to 256) after profiling their
project, or increase `GridCellSize` for exceptionally large structures. Blueprint and C++ systems
can call `RefreshGridValidation` when a game-specific placement rule changes between refreshes.

Owning widgets cancel a specific replicated queue slot through
`ARTSPlayerController::IssueCancelProductionOrder(Actor, QueueIndex, ProductIndex)`. The server
rechecks actor ownership, readiness, component presence, and the live queue index before dispatching
the component's Blueprint Native Event. The bundled info panel renders every authored parallel
queue in a horizontally scrollable strip, updates each queue's progress independently, and maps
every button to its exact queue and product index through this route. Queued-item cancellation works
identically in standalone, listen-server, and remote-client play.

Exact queue contents and timings, the rally target, and the most recently produced actor replicate
only to the owning connection. The native HUD likewise renders production details only for owned
actors, preventing a visible enemy producer from disclosing strategic build information.

## C++ extension points

Override the matching `_Implementation` method in a native component subclass:

```cpp
virtual bool CanAssignBuilder_Implementation(AActor* Builder) const override;
virtual bool BeginConstruction_Implementation(
    TSubclassOf<AActor> BuildingClass,
    const FVector& TargetLocation) override;
virtual bool CanConstructBuildingAt_Implementation(
    TSubclassOf<AActor> BuildingClass,
    const FVector& TargetLocation) const override;
virtual bool CanAssignProduction_Implementation(TSubclassOf<AActor> ProductClass) const override;
virtual int32 FindQueueForProduct_Implementation(TSubclassOf<AActor> ProductClass) const override;
```

Call the event name without `_Implementation` so Unreal dispatches to Blueprint or native logic.

## Cost and authority invariants

The native implementation records the fraction of construction costs actually paid. Cancellation
refunds are based on that paid fraction, so canceling an unstarted or unaffordable site cannot mint
resources. Immediate costs and each multi-resource over-time progress payment are atomic; over-time
costs and progress are capped to the remaining duration so the final frame cannot overcharge.
Refund factors are clamped to `[0,1]`.
Failed immediate-payment starts leave the site unstarted and report false without partially changing
construction state or fabricating a refund.

Production validates supported product classes at the component policy boundary, uses actual
replicated queue counts for indexing, caps final-frame progress/costs, and rejects client-side
mutations. Blueprint and C++ overrides are trusted server rules and must preserve valid classes,
finite times/locations, non-negative costs, queue bounds, ownership, and payment invariants.
