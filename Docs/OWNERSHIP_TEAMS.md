# Ownership and Team Extension Contract

`RTSOwnerComponent` stores gameplay ownership as replicated `RTSPlayerState`, not as a controller
reference. Player state remains network-relevant when a remote controller is not, making it the
stable identity for selection, production, economy, vision, team checks, and observer UI.

`Set Player Owner` accepts an RTS controller, while `Set Player State Owner` supports systems that
already hold replicated identity. Both are authority-only Blueprint/C++ transactions and return
true only when ownership actually changes. Null explicitly clears ownership. Invalid controllers,
controllers without an RTS player state, client mutation, repeated assignments, and cross-world
player states are rejected without altering the current owner.

On commit, the owning actor is awakened for replication and the old/new player-state actor indexes
are updated before `On Owner Changed` is broadcast. The event supplies the new `RTSPlayerState`
directly, so Blueprint code never has to infer identity from a controller that may not replicate to
that connection. Nested ownership mutation from customer callbacks is rejected.

When a system also needs Unreal's network-owner pointer updated, use the game mode's `Transfer
Ownership` transaction. It commits the actor owner and replicated RTS player-state identity as one
operation, rolls the actor owner back if the component rejects the change, applies player advantage
policy only after ownership is stable, and returns whether anything changed. Passing null clears
both identities. Actors without `RTSOwnerComponent` still support engine-only ownership, while an
actor with the component requires an RTS player state for non-null assignment.

Use `Is Same Team As Actor` or `Is Same Team As Controller` for policy checks. These queries require
valid RTS ownership and team membership; neutral, unowned, invalid, and non-RTS actors never become
friendly by accident. C++ systems may retain player states as weak object pointers across deferred
work, but should revalidate the actor, owner component, and player state before committing gameplay.

Team membership is also a server-authoritative transaction. Use `Add To Team` and `Remove From Team`
from Blueprint or C++; both return true only when state changes. Reassignment removes the controller
from its former team index, adds it to the new index, commits the replicated player-state team, and
only then notifies native and Blueprint listeners. Consumers therefore never observe an intermediate
unassigned state. Invalid, repeated, cross-world, client-side, and nested callback mutations are
rejected. Do not mutate the replicated player-state team pointer independently of `RTSTeamInfo`.

Team indices use initial-only replication and may be committed once on authority. Player indices and
observer state are likewise authority-owned, wake the player state for replication when changed, and
report whether a real mutation occurred. This makes UI, caster tools, fog of war, and gameplay policy
safe to extend without guessing whether a request succeeded.
