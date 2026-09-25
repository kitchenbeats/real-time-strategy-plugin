# Economy and Gathering

The default economy supports multiple resource types, finite resource sources, worker carrying
capacity and cooldowns, optional enterable sources, team-owned drop-off buildings, replicated
player wallets, construction/production costs, and Blueprint-assignable change delegates.

## Blueprint extension points

`RTSGathererComponent`, `RTSResourceSourceComponent`, `RTSResourceDrainComponent`, and
`RTSPlayerResourcesComponent` are Blueprintable components. Their rule and mutation functions are
Blueprint Native Events, so a Blueprint component subclass can override only the policy it needs:

- Gatherer availability, preferred/closest source and drain selection, gather range, harvesting,
  and returning resources
- Source admission and extraction
- Drain deposits
- Wallet affordability, additions, individual payments, and atomic multi-resource payments

Call the parent implementation to retain the standard RTS behavior and add project-specific rules
around it. A custom source may implement regeneration or a contested occupancy rule; a custom
gatherer may prefer assigned resource nodes; a custom wallet may apply faction discounts or a
credit limit.

Wallet catalogs and balances replicate only to their owning connection. Runtime catalog changes
drive `On Resource Catalog Changed`, while every committed balance mutation supplies an exact
old/new snapshot through `On Resources Changed`; nested customer callbacks cannot rewrite later
events from an atomic multi-resource payment. Resource nodes replicate both current and maximum
capacity with `On Resources Changed` and `On Capacity Changed`, including extractor pool transfers.
Gatherer payload changes drive `On Carried Resources Changed`, and the active source replicates
owner-only with `On Gathering Source Changed`, so Blueprint HUDs do not need to poll or infer server
state.

For custom source actors, enable actor **Replicates** as well as component replication and include
`RTSVisibleComponent`. A replicated component alone cannot transport a dynamically spawned actor to
remote players. Generator 10 configures actor replication on generated source bases; regenerate
older generated outputs to adopt it. A class you set as a resource source's **Replace With Class**
(`ExistingSourceActorClass`) is yours and is never modified by generation. Preserve the standard visibility routing rather than enabling **Always
Relevant** to work around a missing source. See [Network Security](NETWORK_SECURITY.md).

Extractor absorption runs automatically for pre-placed/finished extractors and when construction
finishes. Custom construction pipelines can call `Try Absorb Raw Source`; it returns true only after
the compatible raw node's exact remaining pool commits and the raw actor is removed. Selection is
deterministic for equidistant nodes, nested absorption is rejected, failed removal rolls the
extractor pool back, and `On Raw Source Absorbed` exposes the committed result to Blueprint.

Extractors require a compatible, non-depleted neutral raw source within their absorption radius
for placement by default. Placement and absorption select the same source; only that source is
excluded from the building collision check. Other buildings and obstacles still block placement.
For a self-contained extractor with its own authored resource pool, disable **Require Raw Source
For Placement** on its component defaults, or call `ConfigureExtractor(Radius, false)` before
`BeginPlay`. Existing one-argument calls use the new required-source default. Code binding an exact
one-argument member-function pointer must adopt the two-argument signature and rebuild.

`Distribute Idle Workers` is an authority-only Blueprint/C++ orchestration call. It requires the
world context and owning controller to belong to the same world, rejects recursive distribution
from order callbacks, and deterministically orders equal-distance workers and sources. Every
worker, controller, source, and resource pool is revalidated after extensible gatherability and
order callbacks, so a callback that removes an actor cannot leave the remaining distribution pass
holding stale gameplay objects.

Harvest start returns whether source admission and required container entry actually committed.
Start, stop, extraction, cargo mutation, and deposit reject callback re-entry. A deposit reserves
cargo before crediting the wallet and restores any amount a custom wallet declines, so failure
cannot duplicate or silently discard a surviving worker's payload. Deposit presentation uses an
unreliable multicast because authoritative cargo and wallet balances already replicate; a burst of
worker drop-offs cannot clog the reliable gameplay channel.

Assign Blueprint component subclasses in the class defaults of your own units, buildings and
controllers, exactly as native components are assigned: in a unit's or building's
**Custom Blueprint**, or in a complete class set through **Replace With Class**. Keep those classes
outside the generated folders. Configure a generated Blueprint through its Content Set entry instead
of editing it. The
built-in gathering order and worker AI resolve these components by base class, so derived components
participate without plugin changes. Configured resource-source actor classes also accept Blueprint
and C++ subclasses, allowing a customer to reskin or specialize a supported source without
duplicating every worker definition.

## C++ extension points

C++ component subclasses override the corresponding `_Implementation` method, for example:

```cpp
virtual bool CanGatherFrom_Implementation(AActor* ResourceSource) const override;
virtual float ExtractResources_Implementation(AActor* Gatherer, float ResourceAmount) override;
virtual bool CanPayResources_Implementation(
    TSubclassOf<URTSResourceType> ResourceType,
    float ResourceAmount) const override;
```

Normal calls use the function name without `_Implementation`; Unreal dispatches them to either the
Blueprint override or the native implementation.

## Authority and invariants

State-changing economy events are authority-only. The native implementation rejects invalid,
negative, zero, and non-finite transfers; validates supported resource types; prevents deposits at
enemy drains; and makes multi-resource payments atomic. Source pools and player wallets replicate
from the server. Mutations wake dormant actors before customer callbacks, source extraction rejects
callback re-entry, and pool-transfer failure leaves the prior pool intact. Client UI should use the
pure queries and replicated delegates for feedback, then request gameplay actions through the
normal controller/order path.

Blueprint and C++ overrides execute as trusted server game rules. They must preserve the same
invariants, validate every live actor and amount, and avoid using client UI state as authority.
