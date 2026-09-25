# Navigation and Flow Fields

RTS pawn controllers use a Blueprint-selectable hybrid policy. The production default is
`Stock Navigation`, which disables controller path replacement entirely and retains Unreal's
navigation path for every pawn. `Military Only` is an experimental opt-in: it preserves stock
navigation for gatherers/builders while military pawns may use the shared field route. `All Pawns`
is also experimental because the sampled grid does not yet model every runtime obstacle or retain
projected navmesh XY. Treat both flow replacement policies as experimental: if you use one, test it
in your own maps with buildings placed during play, many units sent to the same place, mixed unit
sizes and blocked routes.

Generated starter maps use `URTSNavigationSystem` with `ARTSRecastNavMesh`. The Recast class owns a
`DynamicModifiersOnly` runtime-generation default: static floor geometry remains editor-baked while
building footprints and other `bDynamicObstacle`/`NavArea_Null` modifiers rebuild only affected
tiles. This pairing is part of the generated-map contract and does not depend on a customer's
`DefaultEngine.ini`. C++ projects that need a different navigation-data implementation should derive
from `URTSNavigationSystem`, override `ResolveRTSNavigationDataClass`, select that navigation-system
subclass in their map configuration, and preserve an equivalent runtime-modifier policy. Such a
replacement must requalify obstacle insertion/removal, unreachable-goal redirection, path following,
and flow-cache invalidation. Plain blocking collision is not a navigation modifier; runtime blockers
must export an appropriate navigation area.

When explicitly requested through the flow-field APIs, dead ends, malformed data, blocked corners,
allocation limits, and unreachable starts fail closed; they never create a direct segment across an
unverified grid edge. Cache keys include goal height for stacked maps, and navigation rebuilds
invalidate cached fields automatically.

Add `RTSFlowCostVolume` to author impassable regions or expensive terrain. Blueprint and C++ runtime
changes use the authority-only `Configure Flow Cost` transaction, which validates the multiplier and
invalidates existing fields. Editor movement or property edits reconstruct a placed volume and also
invalidate the cache. Invalid cost data is treated as impassable rather than contaminating path costs.

`RTSFlowMovementComponent` is the supported opt-in component-level interface for custom units that
want to follow a field directly. `Move To Goal`
and `Stop Flow Move` are truthful server-authority transactions. A unit holds position while a field
request is generation-budgeted; `On Flow Move Failed` fires after the configured bounded retry window,
and `On Flow Move Finished` fires after the safe effective goal is reached. The component compares its
observed cache epoch before every field read, so navigation rebuilds, navigation-system replacement,
and live flow-cost changes retire retained shared fields before the next movement sample.

Custom C++ consumers that retain the pointer returned by `GetFieldForGoal` must retain the value from
`GetCacheEpoch` at the same time and reacquire before reuse whenever that value changes. Both calls and
`InvalidateAll` are game-thread contracts, matching Unreal's world and navigation APIs. Components that
wrap `RTSFlowMovementComponent` can compare `GetObservedFlowFieldEpoch` for diagnostics without reaching
into plugin-private state.

World-level limits live on `RTSFlowFieldSubsystem` config: cell size, maximum step connection,
generation budget, cache capacity, maximum cells per field, and maximum region half-extent. Invalid or
oversized requests are rejected before allocation. C++ extensions can request shared fields or safe
waypoint paths directly; Blueprint-only projects can use the movement component and cost volumes.
