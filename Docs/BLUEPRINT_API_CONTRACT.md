# Blueprint API Contract

This is the customer-facing contract for extending and operating the Real-Time Strategy plugin
from Blueprint. Blueprint and C++ use the same runtime objects and gameplay rules; Blueprint nodes
do not form a less strict authority or transaction path.

## Finding the right node

All supported Blueprint functions, events, delegates, and editable fields live under an
`RTS|System` category. Start with the system being changed—such as `RTS|Orders`, `RTS|Economy`,
`RTS|Construction`, `RTS|Combat`, `RTS|AI`, or `RTS|UI`—instead of searching a single undifferentiated
RTS list. Every exposed member has an editor tooltip. Customer subclasses should use the same
category tree for their additions.

Pure nodes are side-effect-free queries. Use them for UI refresh, command gating, AI policy, and
validation. Execution-pin nodes either change local presentation, submit a client request, or
perform an authoritative gameplay mutation; their tooltip and authority badge identify which
contract applies.

## Authority and request results

Nodes marked **Authority Only** mutate gameplay truth and must run on the server. The native
implementation still validates authority; the Blueprint badge is guidance, not the security
boundary. From an owning client, submit gameplay intent through the corresponding
`ARTSPlayerController` request instead of calling a component transaction directly.

Every Boolean `Issue...` node labels its output **Request Accepted**. `true` means the request
passed caller-side admission and was submitted (or, when already on authority, was accepted by the
immediate transaction). It does not promise that a networked action will finish: the authoritative
world may reject a stale target or later gameplay may interrupt the action.

Treat replicated state and completion/cancellation/failure delegates as the result of an
asynchronous action. Do not spend resources, show permanent success, or advance objectives merely
because **Request Accepted** is true.

## Failure and feedback

- A pure validation node returns current eligibility only; revalidate on authority immediately
  before mutation.
- A `false` action or request result means no successful local admission or authoritative commit.
  Do not emit a success notification.
- Player-facing rejection text is delivered through the controller error event/delegate and the
  feature-specific failure delegates documented in the system guide.
- Content-authoring failures are reported by **Validate Content Set** and the **RTS Authoring**
  Message Log. Runtime configuration and transaction problems are available through RTS
  diagnostics and match-check output.
- Null actors, invalid classes, non-finite coordinates, invalid indexes, wrong ownership, wrong
  authority, insufficient resources, unmet technology, and stale targets fail without a partial
  gameplay transaction unless a narrower feature contract explicitly documents another result.

## Pins and defaults

Optional queue/product indexes default to the first queue/product where the node advertises a
default of zero. An order index of `-1` means the order's default/no explicit slot only on nodes
that expose that default. Do not invent sentinel values for other pins. World-context pins on
function libraries are supplied by the calling Blueprint when marked as world context.

Arrays and structs returned by getters are snapshots. Changing a returned value does not mutate
plugin state; use the documented action or transaction node. Target actor and location pins are
validated together according to the order or ability target policy.

## Extending in Blueprint

Prefer the narrow `BlueprintNativeEvent` policy hooks documented by each feature. Call the parent
implementation for additive behavior. Skip it only for a documented complete-replacement seam,
and then preserve every authority, ownership, payment, cooldown, replication, notification, and
cleanup invariant named by that seam.

Use `EXTENSION_GUIDE.md` to choose the feature path, `CPP_EXTENSION_CONTRACT.md` for the shared
runtime rules, and the feature guide for its exact events and transaction lifecycle.
