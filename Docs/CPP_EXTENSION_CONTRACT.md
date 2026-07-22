# C++ Runtime Extension Contract

This is the shared contract for project-owned C++ that extends the Real-Time Strategy plugin. It
applies to native subclasses, Blueprint overrides implemented on native classes, delegate
subscribers, subsystem integrations, and asynchronous work. Feature guides add narrower rules but
do not relax this contract.

## Thread model

Plugin gameplay APIs, UObject callbacks, Blueprint events, component delegates, and replication
notifications run on Unreal's game thread unless an API explicitly documents another thread. The
plugin does not expose a worker-thread gameplay mutation seam.

- Read or mutate Actors, Components, UObjects, Worlds, subsystems, replicated containers, and
  delegates only on the game thread.
- Worker tasks may operate on copied value data that has no UObject references. Do not capture raw
  UObject pointers, references to UObject-owned fields, iterators, or views into replicated arrays
  or maps.
- Observe a UObject across a frame or asynchronous boundary with `TWeakObjectPtr`. Marshal the
  result to the game thread, resolve the weak pointer there, and revalidate the world, authority,
  ownership, match state, and transaction preconditions before use.
- Treat arrays and structs returned by Blueprint getters as snapshots unless the API explicitly
  documents a live handle. Never retain an address into a returned or replicated container.

A safe asynchronous calculation copies input values and commits through an authoritative API only
after returning to the game thread:

```cpp
TWeakObjectPtr<ARTSPlayerController> WeakController = Controller;
const FMyRTSInput InputSnapshot = BuildValueSnapshot();

Async(EAsyncExecution::ThreadPool,
    [WeakController, InputSnapshot]()
    {
        const FMyRTSResult Result = ComputeFromValues(InputSnapshot); // No UObject access.

        AsyncTask(ENamedThreads::GameThread,
            [WeakController, Result]()
            {
                ARTSPlayerController* Controller = WeakController.Get();
                if (!IsValid(Controller) || !Controller->HasAuthority())
                {
                    return;
                }

                // Revalidate current ownership, team, visibility, cost, range, cooldown,
                // match state, and target state before calling the supported transaction API.
                CommitValidatedResult(Controller, Result);
            });
    });
```

The project module owns the task and cancellation policy. A task must be safe to finish after its
originating actor, world, match, or map has ended; a weak reference becoming invalid is a normal
cancellation result.

## Authority and transaction boundaries

The server owns gameplay truth. `BlueprintAuthorityOnly` and editor node filtering communicate
intent; they are not a security boundary. Native extensions must enforce authority themselves.

- A client sends intent through a supported request on its owning `ARTSPlayerController`. Never
  trust a client-provided owner, team, price, range result, visibility result, target relationship,
  winner, or arbitrary Actor reference.
- Validate cheap request shape and rate limits first. On authority, resolve authoritative objects
  and revalidate ownership, team, fog visibility, target legality, resources, technology, capacity,
  range, cooldown, match state, and liveness immediately before mutation.
- Use the plugin's transaction APIs for orders, abilities, economy, construction, production,
  research, containment, ownership transfer, elimination, and match completion. Direct field or
  replicated-container edits bypass ordering, rollback, notifications, and security checks.
- A Boolean returned by a direct authority-side transaction reports that immediate authoritative
  commit. A client-side `ARTSPlayerController::Issue...` result reports **Request Accepted**, not
  eventual completion; observe replicated state and feature completion/failure delegates. In both
  cases, propagate `false` instead of broadcasting success or applying durable presentation.
- Delegates and Blueprint events report committed state. Do not use a notification callback as
  authorization, and avoid recursively starting the same transaction from its completion delegate.

Client prediction may drive disposable presentation only. Replicated state and authoritative
transaction results must correct it. An unreliable multicast can communicate effects, never durable
gameplay truth.

## UObject ownership and lifetime

Use Unreal's ownership and garbage-collection model explicitly:

- Store owned or required UObject references in reflected fields (`UPROPERTY` with `TObjectPtr`, or
  the equivalent reflected container). Use `TWeakObjectPtr` for observed actors and components that
  may disappear independently.
- A raw pointer is acceptable only as a short-lived local on the game thread. Do not retain it
  across ticks, timers, latent actions, delegate callbacks, async tasks, travel, or match restart.
- World subsystems are world-scoped. Resolve them from the current world; do not cache them across
  seamless travel, map load, PIE instance changes, or world teardown.
- Reject cross-world objects. Recheck `GetWorld()` when an object supplied by external project code
  reaches a transaction boundary.
- Do not mutate a Class Default Object at runtime. Configure instance defaults in constructors or
  supported setup APIs before the object registers or begins play.

Bind lifecycle-aware delegates to UObjects. If the publisher can outlive the subscriber, remove the
binding during the subscriber's teardown (`NativeDestruct`, `EndPlay`, or `Deinitialize`, as
appropriate). Clear owned timers, latent work, and external registrations during teardown. Cleanup
must tolerate partial initialization and repeated or reordered world shutdown.

Replication notifications can arrive before optional presentation objects exist and during
teardown. They must be idempotent, tolerate missing dependencies, and derive presentation from the
current replicated state rather than assuming notification order.

## Override rules

- Implement a `BlueprintNativeEvent` in C++ through its `_Implementation` method.
- Call the parent implementation for additive policy. Skip it only where the feature guide names
  the event as a complete replacement seam and the subclass assumes every documented invariant.
- Query and policy callbacks return decisions; they do not mutate gameplay, emit success, or launch
  asynchronous work unless their feature contract explicitly says otherwise.
- Transaction overrides preserve the documented validation, payment, cooldown, refund, replication,
  delegate, and cleanup order. Prefer overriding the narrow policy hook instead of replacing the
  transaction.
- Never call a Blueprint event from a worker thread. A C++ override and its Blueprint implementation
  share the same game-thread, authority, and lifetime rules.

## Extension review checklist

Before shipping a project-owned extension, verify that it:

1. Compiles in Editor, Development, and Shipping without unity or shared-PCH assumptions.
2. Mutates gameplay only on authority and routes client intent through the owning controller.
3. Revalidates every security and transaction precondition immediately before commit.
4. Uses supported transaction APIs and treats their return value as truth.
5. Holds UObject state through reflected strong references or deliberate weak references.
6. Unbinds external delegates and clears timers/tasks/registrations during teardown.
7. Touches UObjects and Blueprint only on the game thread; async work consumes copied values.
8. Handles destroyed actors, world teardown, travel, late replication, and missing presentation.
9. Preserves the parent contract for every override and documents intentional replacement seams.
10. Passes the Content Set validator, relevant automation, and a packaged multiplayer run.

See `NETWORK_SECURITY.md` for hostile-client boundaries and fog-of-war replication,
`OWNERSHIP_TEAMS.md` for ownership and team identity, and each feature guide for its transaction's
specific invariants.
