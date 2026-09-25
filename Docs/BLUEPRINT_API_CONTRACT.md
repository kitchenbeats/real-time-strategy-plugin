# Blueprint API Rules

These are the rules every Real-Time Strategy Blueprint node follows. Blueprint and C++ use the same
runtime objects and gameplay rules; Blueprint nodes are not a less strict path. For the most useful
events and nodes with examples, start with the [Blueprint Quick Reference](BLUEPRINT_QUICK_REFERENCE.md).

## Finding the right node

All supported Blueprint functions, events, delegates, and editable fields live under an
`RTS|System` category. Start with the system being changed—such as `RTS|Orders`, `RTS|Economy`,
`RTS|Construction`, `RTS|Combat`, `RTS|AI`, or `RTS|UI Data`—instead of searching a single
undifferentiated RTS list. Every exposed member has an editor tooltip. Your own subclasses can use
the same category tree for their additions.

Pure nodes are side-effect-free queries. Use them for UI refresh, command gating, AI policy, and
validation. Execution-pin nodes either change local presentation, submit a client request, or
perform an authoritative gameplay mutation; their tooltip and authority badge identify which
contract applies.

## Authority and request results

Nodes marked **Authority Only** mutate gameplay truth and must run on the server. The native
implementation still validates authority; the Blueprint badge is guidance, not the security
boundary. From an owning client, use the corresponding high-level request on
`ARTSPlayerController` instead of calling an authority-only gameplay transaction directly. For a
generic selected-unit order, call **Get Order Submission** on the owning controller and then
**Issue Order to Selected Actors** on its replicated native Order Submission component. That
controller-owned component is a supported request boundary; do not create a second one. The
controller remains the high-level command, order-class policy, targeting, and event surface.

Boolean request nodes label their output **Request Accepted** or **Request Submitted**. `true`
means the request passed caller-side admission and was submitted. It does not report eventual
completion or, for a selected-unit order, how many units authority accepted: the server may reject
malformed or stale input, and later gameplay may interrupt an accepted action.

Treat replicated state and completion/cancellation/failure delegates as the result of an
asynchronous action. Do not spend resources, show permanent success, or advance objectives merely
because a request result is true.

## Failure and feedback

- A pure validation node returns current eligibility only; revalidate on authority immediately
  before mutation.
- A `false` action or request result means no successful local admission or authoritative commit.
  Do not emit a success notification.
- Player-facing rejection text is delivered through the controller error event/delegate and the
  feature-specific failure delegates documented in the system guide.
- Content-authoring failures are reported by **Check for Problems** and the **RTS Authoring**
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
implementation for additive behavior. Skip it only where a guide says the function is a complete
replacement, and then keep every authority, ownership, payment, cooldown, replication, notification
and cleanup rule that guide names.

Use [Extending with C++](EXTENSION_GUIDE.md) to choose the feature path,
[C++ Extension Rules](CPP_EXTENSION_CONTRACT.md) for the shared runtime rules, and the feature guide
for its exact events and transaction order.
