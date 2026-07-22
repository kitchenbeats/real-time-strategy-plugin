# Multiplayer Vision Security

The plugin ships with connection-aware actor replication for projects using `ARTSGameMode` and
Unreal's legacy networking stack. It is enabled by default. A replicated actor that carries a
`URTSVisibleComponent` is sent only to connections whose authoritative server-side vision state is
`Visible`.

This keeps live enemy position, health, orders, construction state, and attached replicated
components off clients that cannot currently see the actor. When an enemy leaves vision, its actor
channel is closed. The existing vision system can continue to display last-known client-side ghost
data without receiving hidden updates.

## Default policy

- Currently visible actors replicate to that viewer.
- A player's owned pawn and actors owned directly by its controller always replicate to that
  player.
- Observer player states and replay connections receive complete state.
- `Known` means last-known information only; it does not authorize live actor replication.
- Replicated actors without `URTSVisibleComponent` use normal spatial, always-relevant, or
  owner-only routing.

Add `URTSVisibleComponent` to every replicated unit, building, resource, projectile, or other actor
whose live existence or state must be protected by fog of war. The component and its vision state
must remain server-authoritative.

Native `RTS Projectile` actors already include both `RTS Owner Component` and `RTS Visible
Component`, and Blueprint subclasses inherit that secure default. The firing controller owns the
projectile for its own connection while every other connection is admitted only by authoritative
`Visible` state; `Known` does not expose a live projectile. A shot is presented from one immutable
replicated initialization snapshot rather than a reliable multicast or replicated transform stream.
This keeps projectile traffic bounded and preserves late-relevance catch-up while impact and damage
remain server-authoritative. A replacement projectile class must preserve both components and the
same per-connection relevance policy.

## Automatic setup

`ARTSGameMode` installs `URTSReplicationGraph` when its server NetDriver has no replication driver.
The `bAutoInstallSecureReplicationGraph` class default is enabled, so Blueprint-only projects get
the secure policy automatically by deriving their game mode from `ARTSGameMode`.

If a project supplies a custom Replication Graph, set
`bAutoInstallSecureReplicationGraph` to false and implement an equivalent per-connection filter for
all `URTSVisibleComponent` actors. The plugin deliberately does not replace a project's existing
driver. It logs a warning so an insecure integration cannot go unnoticed.

## Iris

For Unreal Engine 5.8, the included filter targets the Replication Graph networking path. Iris
cannot host a Replication Graph driver. If Iris is enabled, the plugin logs an error at server
startup; the project must provide an Iris connection filter with the same policy or disable Iris.
Do not ship a competitive multiplayer build with that error unresolved.

## Owner-private strategic state

Fog relevance determines whether an actor exists on a connection; per-property conditions protect
details even while that actor is visible. Production queue classes and timings, rally targets, and
the most recently produced actor replicate only to the producer's owning connection. Container
passenger identities are likewise owner-only, with a separate public occupancy count. Replacement
components should preserve these conditions unless disclosing the data is an explicit game rule.

## Starter participant lifecycle

The generated skirmish can require `MinimumHumanPlayersToStart` before creating bases. The
authority sorts connected player controllers by stable player id, assigns the first humans to
distinct base/team slots, fills unclaimed bases with AI, and places excess participants in observer
mode. A player joining after the match has started is also an observer; the starter does not grant
late clients control of an established base. Projects that implement reconnect ownership or base
takeover should replace this policy explicitly and revalidate ownership, vision, and private state
before granting control.

The automatic configuration audit runs only in standalone or authoritative server worlds.
Navigation data, AI controllers, and authoritative wallets are intentionally absent or incomplete
on network clients, so interpreting those client replicas as server configuration would create
false failures and unsafe client-side repair attempts.

## Release verification

The replication policy is implemented and focused-contract-tested, but the packaged dedicated
privacy/hostile-client matrix remains open. The checks below are required before describing hidden
state as commercially qualified.

Test the packaged game with a dedicated server and at least two independent clients. Verify that:

1. An unseen enemy actor does not exist in the receiving client's replicated world.
2. Entering vision creates the authoritative actor with current state.
3. Leaving vision removes the authoritative actor and stops all live updates.
4. Last-known UI or ghosts contain only the intentionally captured snapshot.
5. Re-entering vision restores the actor with current authoritative state.
6. Spectators and replays behave according to the intended game rules.

Treat UI concealment, hidden meshes, and client-side `IsNetRelevantFor` overrides as presentation
only. Competitive fog-of-war security requires server-side, connection-aware replication filtering
like the policy above.

## Inbound client requests

The server also validates the plugin's client-issued gameplay requests. Player chat accepts only
All and Team channels, enforces a hard payload cap, and uses a per-player token bucket configured by
`MessagesPerSecond` and `MessageBurstLimit`. System chat remains authority-only.

Player map pings follow the same authoritative shape: the owning controller's `RTSPingComponent`
accepts only bounded finite coordinates and capped text, sanitizes the label, enforces a per-player
token bucket, and routes only through each recipient controller's server-side
`CanReceivePlayerPing` policy. The default reaches teammates and observers; active teamed players
are the only default senders. Override `CanSendPlayerPing` or `CanReceivePlayerPing` in a controller
Blueprint/C++ class for alliances or custom caster rules without moving trust to the client.

Order requests require an owned pawn, a finite target location, and a server-approved order class.
The default `ARTSPlayerController` policy accepts its `DefaultOrders` plus explicit construction and
stop commands. When adding a custom client-issued order in Blueprint, override
`IsOrderClassAllowedFromClient`; in C++, override
`IsOrderClassAllowedFromClient_Implementation`. Keep live resource, cooldown, target, and technology
checks in the order/component implementation so normal state changes during network transit cause a
safe rejection rather than disconnecting the player.

Selected-unit orders are capped at 1,024 unique owned pawns. Commands above 32 pawns use one bounded
begin request plus ordered chunks of at most 32 pawn references. Authority retains only one incomplete
transaction per controller, reserves the complete chunk work budget before allocating it, rejects
replayed begin IDs at RPC validation, and executes nothing until the complete batch validates.
Reliable rejection completions are also admission-gated: an inbound request that cannot obtain a
response work unit is dropped without reflecting traffic back onto the reliable channel. Chunks from
a rate-denied begin are not treated as prepaid and must pass the shared budget individually.

These controls bound application state, gameplay work, and server-generated responses for admitted
requests. They do not claim that an application token bucket can prevent a modified client from
putting reliable RPCs into Unreal's connection-level transport before dispatch. Source automation
uses the generated authoritative RPC thunk plus adversarial validator/budget cases; it is not a
packaged hostile-client saturation test. Packet-loss delivery, queued-bunch growth, connection
closure, and reliable-buffer behavior remain mandatory packaged qualification under `SEC-003` and
`SEC-004`.

Begin-construction requests additionally resolve the builder's indexed class allowlist and execute
its authoritative `CanConstructBuildingAt` policy during order admission. The builder repeats that
policy immediately before spawning, using live shape or authored-footprint overlap state, so a
client preview or an AI planner cannot reserve stale authority over an occupied location.

Ability, stance, research, construction, production, and container requests likewise require an
actor owned directly by the requesting controller. Indexed production cancellation bounds its queue
and product indices at the RPC boundary, then rechecks the live replicated queue before mutation.
Research cancellation rechecks that research is still active and refunds only the authority-side
payment captured when it began. Stale gameplay state is dropped without treating normal latency as
a connection-level validation failure.
