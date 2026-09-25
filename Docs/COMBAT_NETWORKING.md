# Combat and Networking

Combat state is server-authoritative. Health, shields, energy, ability cooldowns, attack-cooldown
deadlines, stance, upgrades, and kills replicate to clients; clients use the replicated values for
HUD and presentation. Runtime initialization happens only on authority so a component's `BeginPlay`
cannot overwrite an initial replication payload.

`RTSAttackComponent` publishes one compact presentation event whenever an authoritative attack is
used. Relevant clients receive `OnAttackUsed`, including the authored attack, target, and projectile
reference, so animation and presentation hooks run consistently for remote units. Running weapon
cooldowns replicate only when an attack starts as synchronized server-time deadlines. Clients derive
smooth values through `GetRemainingCooldownForAttack` or `GetRemainingCooldownFraction` without
per-frame network updates, and emit `OnCooldownReady` once when the local synchronized clock reaches
the deadline.

### Attack presentation origins

Every `RTS Attack Data` entry has its own `Origin Socket` and `Origin Offset`, so ground and air
weapons, dual cannons, turrets, stance-specific weapons, and melee swings can use different
presentation points. The
contract is presentation-vendor neutral:

- Leave `Origin Socket` empty for greyboxes, static/skeletal reskins without sockets, and code-only
  actors. `Origin Offset` is then transformed from actor-local space.
- Name a socket or skeletal bone to launch from that mesh location. `Origin Offset` is transformed
  from the socket's local space and therefore follows the fitted presentation.
- An explicitly named missing socket rejects an authoritative projectile transaction before spawn,
  cooldown, replicated attack publication, or callbacks. Melee gameplay remains authoritative and
  independent of presentation, so a missing melee socket suppresses only the built-in FX. Content
  Set validation rejects either configuration with the exact unit and attack index before generation.

Projectile hit and damage truth still comes from the authoritative target and attack data; evaluated
animation pose is not used to decide whether the attack hits. The socket controls launch presentation,
not combat legality. The built-in melee FX uses the same origin contract, attaches to named sockets,
spawns on relevant clients from the replicated attack event, and skips dedicated-server rendering.

### Incoming projectile target points

Add `RTS Projectile Target Component` when incoming shots should aim somewhere other than the actor
origin. Its `Target Points` array is equally authorable in component Details, Blueprint, and C++:

- An empty array intentionally targets the actor origin and works for greyboxes and code-only actors.
- A point with only `Offset` uses actor-local space. Add `Mesh Component Tag` to make it relative to
  a particular mesh component, including static meshes without sockets.
- Add `Socket` to resolve a socket or skeletal bone. The optional component tag disambiguates actors
  with multiple meshes; the offset is then socket-local.
- Invalid offsets, missing tags, and missing named sockets fail the authoritative projectile
  transaction before cooldown or attack publication. They never silently redirect a shot to the
  actor's feet.

The server chooses one point and records that exact world location in the projectile's immutable,
one-shot replicated fire state, so every relevant client renders the same trajectory and impact.
This is bounded state replication, not a reliable multicast per shot: a connection receives at most
the latest initial snapshot when the secure vision policy makes that projectile relevant. The
snapshot also carries its authoritative origin and synchronized fire time, allowing a client that
learns about an in-flight projectile later to catch its local presentation up without continuously
replicating movement. Damage, hit count, splash resolution, and detonation remain authority-only.
Meshes are resolved when each shot is admitted; runtime mesh swaps cannot leave a stale cached
component behind.

## Player-issued actions

Do not call an authority-only gameplay component directly from a client widget. Selected-unit orders
go through the replicated, client-owned `URTSOrderSubmissionComponent` returned by
`ARTSPlayerController::GetOrderSubmission()`:

- `Issue Order to Selected Actors`

Other player actions continue through the owning `RTSPlayerController` request nodes:

- `Issue Ability Order`
- `Issue Research Order`
- `Issue Cancel Research Order`
- `Issue Cancel Production Order` (queue and product index)
- `Issue Stance Order`
- `Issue Cancel Construction Order`
- `Issue Unload Actor Order`
- `Issue Unload All Order`

The bundled command card already uses these paths. Each server request verifies that the gameplay
actor belongs to the requesting controller. Ability requests also validate the index, finite target
data, target type, current visibility for unit targets, cooldown, range, and energy on the server.
Transient failures such as a cooldown changing while the request is in flight are dropped without
disconnecting the player.

Selected-unit orders use packet-safe batches instead of one RPC per pawn. The common one-to-32-unit
command is one self-contained reliable RPC. Larger commands use a bounded application-level
transaction: a constant-size begin request reserves the complete work budget, followed by ordered
reliable chunks of at most 32 pawn references so every RPC remains below UE's non-fragmenting
single-bunch envelope.
Authority retains at most one transaction and 1,024 pawn references per controller for five seconds,
rejects replayed IDs, missing/out-of-order/oversized chunks, duplicates, foreign ownership, and
malformed data, and autonomously clears stale state against an undilated wall clock. It executes
nothing until the complete set passes validation. A normal 1,000-unit
command therefore uses 32 packet-safe chunks and 32 work units. Every strategic server request shares
the `URTSOrderSubmissionComponent` budget, including retained ability, stance, research,
construction, production, container, and surrender RPCs on the controller. The component exposes
rate-admitted and rate-rejected work-unit counters for security, network, and soak evidence;
admission happens before live gameplay validation and is not reported as successful gameplay.
Replayed begin requests fail the network validator closed. Defense-in-depth rejection paths never
reflect a reliable completion unless at least one response work unit was admitted; a rate-denied
begin grants no "prepaid" allowance to chunks already present in the reliable stream, so those chunks
must pass the budget individually. Remote owning clients
receive `OnSubmittedOrderBatch` immediately for predicted UI, while `OnIssuedOrder` is reserved for
actual authority issuance. A reliable authority completion releases client flow control, which caps
large outstanding group-order transactions at one (at most 33 reliable RPCs) rather than allowing
rapid 1,000-unit commands to approach Unreal's 512-bunch reliable-channel ceiling. Commands of 32 or
fewer units do not occupy that transaction slot. After a bounded wall-clock deadline, the owning
client surfaces a missing-acknowledgement error once but deliberately retains the slot: reliable RPCs
cannot be cancelled, so only the server acknowledgement or connection/controller teardown may
release it without permitting stalled-channel backlog growth. Projects may tune the inherited order
component's sustained and burst defaults, but hard ceilings remain enforced in code.

These batching, replay, and rate-budget contracts are covered by validator and adversarial-budget
automation plus a real in-process, two-world listen-server PIE proof. That proof sends an actual
32-reference pawn-array RPC from the owning client's replicated order component, using repeated
references to one already-mapped pawn rather than 32 distinct object exports. It admits a transaction
on the authority component, expires it through the remote controller's independent component tick,
and receives the reliable completion on the client component to release the exact flow-control slot.
It verifies bidirectional component RPC routing and leaves both owning connections open.

The PIE proof does not emulate a modified remote client or prove Unreal's transport queues under
packet loss and saturation. Unreal reliable transport is connection-bounded, but the application
token bucket runs only after an RPC is delivered and deserialized. The plugin has not yet been
tested against modified clients flooding a packaged server, so it makes no claim about resisting
that kind of abuse. Test your packaged game on each platform and network setup you ship.

Container contents are server-authoritative. `GetOccupancy` replicates to every relevant client,
while the full `GetContainedActors` identity list replicates only to the container's owning
connection so hidden passengers are not disclosed to opponents. Owning client widgets unload through
the two controller request nodes above; the server verifies ownership and current membership again
before mutating the container. The bundled info panel is already wired to this contract: it shows
owner-private passenger portraits with click-to-unload and an unload-all control, while opponents
and observers see only neutral occupied slots and the public count. Large manifests are visually
bounded without hiding the exact occupancy count.
Runtime capacity changes reject values below current occupancy; they never evict or orphan passengers.

The bundled command card calls `BeginAbilityTargeting` for unit and ground abilities. It supplies a
crosshair cursor, confirm/cancel prompt, caster-centered range ring, left-click confirmation,
right-click cancellation, invalid-target retry, and minimap support. A replacement Blueprint or C++
UI can bind `OnTargetingStarted` / `OnTargetingEnded` and call `ConfirmTargetingAt`, or bypass the
interaction layer and call `IssueAbilityOrder` with a unit target or ground location. Self abilities
are normalized to the caster on the server, so client-supplied target data cannot redirect them.

## Extension points

`RTSAttackComponent` and `RTSStanceComponent` are Blueprintable. Override their Blueprint Native
Events for project rules:

- `Can Use Attack` and `Use Attack`
- `Can Set Stance` and `Set Stance`

Call the parent implementation when retaining the standard cooldown, damage/projectile, replicated
attack presentation, stance, immobilization-tag, diagnostics, FX, and delegate behavior. In C++,
override the matching `_Implementation` method.

`RTSAbilityComponent`, `RTSEnergyComponent`, `RTSHealthComponent`, `RTSResearchComponent`, and
`RTSUpgradeComponent` are also Blueprintable. Research cost, duration, admission, start, and cancel
are Blueprint Native Events with native defaults. Canceling returns only the exact payment captured
at start, multiplied by the option's clamped `Cancel Refund Factor`; start/cancel/refund delegates
support Blueprint presentation without polling. State-changing nodes are authority-only; use
controller requests for player actions and call direct mutation nodes only in trusted server logic.
Player upgrade snapshots can be authored or replaced with `Configure Upgrade Levels`; keys must be
non-empty and unique and levels non-negative. The entire snapshot replicates together, including
key removals, and local-client UI receives exact old/new level notifications plus
`On Upgrade Levels Configured`. Direct set/increment calls reject re-entry from customer callbacks,
wake the owning controller for replication, and report whether a level actually committed.

Production rally targets are owner-private strategic state. Actor/location setters and clear report
whether the authoritative change committed, reject invalid or re-entrant requests, wake dormant
producers, and drive `On Rally Point Changed` on authority and the owning client. Actor rallies bind
their target lifecycle and clear automatically when that target is destroyed, so a later product can
never inherit a stale actor reference or accidental origin fallback.

### Runtime unit factories

Blueprint and C++ factories can configure every durability setting through one transaction. Build an
`RTS Health Profile` struct and pass it to `Configure Health Profile` on authority. The node rejects
the entire profile if a maximum, regeneration rate, reaction radius, armor value, or enum is invalid;
it never applies a partial configuration. `Get Health Profile` returns the committed settings.

```cpp
FRTSHealthProfile Profile;
Profile.MaximumHealth = 240.0f;
Profile.Armor = 3;
Profile.MaximumShield = 80.0f;
Profile.bRegenerateShield = true;
Profile.ShieldRegenerationRate = 2.0f;
Profile.ActorDeathType = ERTSActorDeathType::DEATH_Destroy;

if (!HealthComponent->ConfigureHealthProfile(Profile))
{
	// The caller is not authoritative, the actor is dead/in a health transaction, or Profile is invalid.
}
```

Configure components before play when constructing a unit; `BeginPlay` initializes health and shield
to the configured maxima. Reconfiguring a live unit preserves its current absolute pool values and
clamps them to the new maxima; a pool that was full remains full. Runtime profiles replicate as one
snapshot, and `On Health Profile Changed` lets Blueprint or C++ presentation refresh without polling.
Class-default properties remain supported for ordinary Blueprint-authored units, so a Blueprint-only
customer can still duplicate a supplied unit, change defaults in Details, reskin it, and play.

Runtime spellcasters use the parallel `RTS Energy Profile` and `Configure Energy Profile` API for
maximum energy, starting energy, regeneration rate, and tick interval. The profile is validated and
replicated atomically. Configure it before play to initialize the requested starting pool, or apply it
to a live caster to preserve/clamp current energy while replacing the regeneration timer safely.
`On Energy Profile Changed` and `On Energy Changed` provide separate presentation hooks for settings
and runtime pool changes. Direct energy writes and spends return success/failure, are authority-only,
wake dormant actors for prompt replication, and reject re-entrant mutation from customer callbacks.
Direct health and shield writes follow the same truthful authority contract. Health, death, gameplay
tags, kill credit, and elimination bookkeeping commit before customer health/death callbacks, so a
listener never observes a zero-health actor that still reports itself alive.

`Death Destroy Delay Seconds` controls corpse-presentation lifetime for `Destroy` death behavior and
is available in health profiles plus generated unit/building definitions. On the authoritative death
transition, movement and AI stop and all interactive gameplay components—including selection,
collision, attacks, production, construction, economy, supply, and vision—retire immediately. The
actor retains identity, health, and presentation only until the server lifespan expires; clients also
clear local hover/selection and collision when death replication arrives. A value of zero explicitly
destroys immediately. Skeletal Content Sets reject a delay shorter than their longest configured
death clip. `Do Nothing` remains the revivable policy: revival clears the terminal animation state and
restores the exact movement mode saved before death.

Runtime transform/siege units use `RTS Stance Catalog` and `Configure Stances`. Each stance name must
be non-empty and unique, the optional default must name a catalog entry, and every nested attack is
checked by the same `Is Attack Data Valid` rules used by the attack component. The complete catalog
replicates atomically before active-stance presentation. A live catalog may be replaced until the
owner commits its first attack; afterward it is immutable so replicated stance names and attack
indices can never resolve against different weapon data. `Set Stance`, `Toggle Siege`, and local
authoritative `Issue Stance Order` report whether a transition actually committed. Override `Can Set
Stance` for project policy, and call the parent `Set Stance` implementation to retain replicated
state, immobilization tags, forced network updates, and `On Stance Changed` delivery.

`RTSContainerComponent` and `RTSContainableComponent` are Blueprintable. Override `Can Load Actor`,
`Load Actor`, `Unload Actor`, or `Unload All` for cargo size, boarding, ejection, bunker, or transport
rules. Call the parent mutation to retain the owner-private occupant list, public occupancy count,
containment visibility/collision state, moving-transport attachment, destruction cleanup, delegates,
and forced network update. Override `Get Unload Location` for doors, ramps, dropship sockets, or
project-specific ejection layouts; the native default distributes passengers outside the combined
collision radii and projects the result to navigation.

Invalid or non-finite authored runtime values are rejected before they can corrupt replicated state.
Health and shield writes clamp to their authored maxima, energy costs cannot be negative or NaN,
attack/projectile parameters are validated, and repeated writes to an already-dead actor do not fire
the death transition again.
