# AI and Match Policies

The built-in bot is faction-agnostic: it discovers workers, production, supply, extractors, research,
abilities, and army buildings from the player's own actor/component data. A reskin or data-only
faction normally uses it unchanged. Projects can add a strategic twist or replace it without
forking `RTSAIBotSubsystem`.

## Built-in strategic safety policy

Attack squads and each unit's close-range automatic acquisition choose targets by capability, not
actor name or faction class: armed combat threats first, then production buildings and construction
sites, then workers, then all other objectives. Distance breaks ties within a tier. This keeps units
responsive to immediate threats while preventing a winning army from chasing an indefinitely
replenished worker stream and leaving the structure producing it alive.

Mobile squads require a working movement component, a movable updated component and a finite
positive native maximum speed. Stationary armed structures remain local defenders and receive no
squad march or distant chase orders. Deployed artillery retains its locomotion capability so normal
squad stance handling can undeploy it. Units that lose movement capability leave squad membership;
ready units can rejoin when mobility returns.

The bot discovers defensive structures through working weapon capabilities in its workers'
construction catalogs. After the economic workforce, a ready resource drain and a first military
producer are established, it can save for one home defense. Completed sites, unfinished sites and
admitted builder travel share that cap. Essential blocking construction retains priority. Placement
must cover the economic anchor and pass native footprint, worker-corridor, navigation and threat
checks. An infeasible optional defense releases its savings; an admitted unpaid site keeps its
normal reservation. Renamed or multi-role armed buildings use the same policy.

Production preserves worker recovery, construction savings, supply, affordability and technology
checks. Among admitted nonworker products, it first fills missing observed combat coverage, then
bounded healing support, then other useful capabilities. A unit's positive-damage weapons and
authored stances determine whether it can answer ground, air or both; its name and visual mesh do
not. Living owned units and every paid production queue count together, so two producers cannot
both fill the same remaining allowance. Cancelled products and owned casualties release it.

This composition heuristic is original project policy, not a copied upstream strategy. The recent
observed air share sets desired air coverage, with one ground observation as a minimum baseline
for unknown opposition and structure assaults. Generalists count once as army and cover both
domains. Dedicated air-only products are purchased only below that coverage target. Targeted,
positive healing support is limited to one healer per four armed nonworkers; armed healers also
credit that allowance. Products whose custom capabilities are not understood retain the previous
fallback selection behavior. Random selection breaks ties among equally useful affordable choices.
If only an unnecessary specialist is affordable, its producer keeps the resources for a useful
product. Building acquisition priorities and payment transactions are unchanged.

The bot records combat domains only while hostile units are visible, then retains those captured
facts for the existing 60-second tactical memory horizon. It does not query a hidden actor's later
weapon changes or infer an unseen death from object lifetime. Allied units create no hostile demand.
The engineering references are CommandCenter's observed unit information and separate ground/air
defender demand, and python-sc2's visible-domain filtering and owned-plus-pending count gates.
These choices improve useful composition; they do not guarantee every authored unit is trained in
every match. Projects needing specific compositions can replace the strategy through the existing
controller hooks below.

The economy manager defers a new expansion when a hostile combat force controls either the proposed
site or any existing townhall region. Each region is evaluated independently so a defended home base
cannot hide an occupied remote expansion. Small patrols do not halt macro play, allied team units
count as local defenders, and construction resumes after the decisive threat leaves. Extractor,
supply, production, and technology construction uses the same last-moment check at its validated
build location, preventing builders from feeding an occupied base with exposed replacement
structures. During strategic collapse it also compares a decisive occupation at any owned building
against the team's total surviving combat strength, so moving a replacement foundation just beyond
one tactical radius cannot create an endless base-trade loop. Projects can rebalance that policy in
`DefaultGame.ini` without subclassing:

```ini
[/Script/RealTimeStrategy.RTSAIBotSubsystem]
ExpansionThreatRadius=2500.0
ExpansionThreatMinimumCombatants=4
ExpansionThreatAdvantageRatio=2.0
```

The ratio is hostile combatants divided by friendly and allied combatants in the same region. With
the defaults, at least four hostiles and a 2:1 local advantage are required. Keep the minimum above
normal scout-party size and tune the radius to the actual footprint of a base; permissive values
make the AI economically more reckless by design.

Low-workforce recovery also changes harvesting priorities. While living workers plus queued
replacements remain below `MinimumWorkersForTech` (clamped to `WorkerTargetPerPlayer`), the bot
reads the unmet resource costs of workers available from its ready production structures. It can
move an empty-handed miner from an unrelated resource to an available source of a missing cost.
The new gather order must pass normal admission before the old source slot is released. Carriers,
construction workers, reserved scouts and drafted defenders keep their jobs; source capacity is
never exceeded. If the required resource has no available slot, the miner keeps its productive
assignment. If the required resource is unavailable locally, idle workers can still use other
available resources. Once a worker is affordable, the recovery override stops. This policy uses
authored gathering capabilities, prerequisites and costs, without resource or faction names.

Worker defense is anchored to ready, living resource-dropoff bases. A forward Barracks does not
extend the area from which economic workers are recruited. Only visible, living mobile attackers
with a usable ground attack create worker-defense demand; stationary structures and unarmed
actors remain strategic targets rather than worker-rush intruders. Local military reduces the
existing two-defenders-per-armed-unit / one-per-worker quota. Demand and relief are matched per
legal worker attack target: airborne or invulnerable enemies cannot inflate a ground-worker quota,
and air-only defenders cannot reduce it. Stationary defenders relieve workers only when their
compatible attack can actually reach that intruder. There is no fixed worker draft floor.

Each drafted worker retains its economic anchor and issued target. When military relief arrives,
a target leaves, or the defender crosses the 2,500 cm economic-region boundary, excess or invalid
drafts are released. Release cancels only the exact defense attack issued by this subsystem;
intervening custom orders remain intact. A worker carrying resources enters the normal Return
Resources order when a compatible drain exists. Idle/mining workers can be drafted; carriers,
construction orders, extractor crews and caller-reserved scouts keep their jobs. Scouts likewise
do not take an attack-ordered defender and retreat toward the player's primary resource drain.

C++ strategies calling `FRTSSquadSystem::TickSquads` may pass a fifth argument containing protected
worker weak pointers. The original four-argument overload remains supported and means no external
reservations. The built-in coordinator supplies its active scouts. Both overloads execute real
orders and belong in authority-owned strategy code.

## Scout observation and cleared regions

Scout discovery, worker harassment and nearby-worker danger checks require a valid hostile team
and current visibility for the scouting player's controller. Allied units and structures are never
hostile discoveries, and server knowledge of hidden actors does not supply scouting information.

A discovered building position is an active scouting destination, not an everlasting orbit. When
that remembered tile is currently visible to the scouting team, the scout keeps or retargets the
region only if a living hostile building is currently observed there. Otherwise it resumes normal
candidate exploration. Destruction, ownership changes or invalid actor references in fog cannot
clear the remembered destination; the system does not inspect hidden actor liveness. Existing
builder, carried-resource and defense-worker reservations still apply.

## Blueprint AI policy

1. Create a Blueprint derived from `RTSPlayerAIController` and select it in the skirmish definition
   or GameMode.
2. Leave **Use Built In Strategy** enabled when the custom policy is additive. Disable it when the
   Blueprint owns all economy, production, construction, research, ability, scouting, and combat
   decisions.
3. Override **Execute Strategy Think**. The plugin calls it on server authority at the built-in
   strategy interval; it is not a frame tick.
4. Use **Get Owned Actors** for the maintained roster snapshot. Filter invalid or capability-missing
   actors before issuing decisions, and use the component/controller transaction APIs for changes.

A minimal replacement opening can call **Get Next Pawn To Produce**, reject the `Pawn` sentinel,
check **Meets All Requirements For** and **Can Pay For**, then call **Start Production**. That is a
complete server decision through the same payment and queue validation used by the built-in bot.
More advanced policies can inspect production, builder, research, ability, health, supply, and order
components on the owned roster.

`Execute Strategy Think` also runs after the built-in managers when they remain enabled. Use that
mode for bounded additions such as a faction-specific research preference or an objective response.
Do not reissue the same order every heartbeat: compare current orders/state and change intent only
when the decision changes.

## C++ AI policy

Derive from `ARTSPlayerAIController` and override the same two functions:

```cpp
bool AMyOpeningAIController::ShouldUseBuiltInStrategy_Implementation() const
{
	return false;
}

void AMyOpeningAIController::ExecuteStrategyThink_Implementation(float DeltaSeconds)
{
	const TSubclassOf<APawn> Next = GetNextPawnToProduce();
	if (Next && Next != APawn::StaticClass() &&
		MeetsAllRequirementsFor(Next) && CanPayFor(Next))
	{
		StartProduction(Next);
	}
}
```

The callback is authority-only and rate-limited by `URTSAIBotSubsystem`. `GetOwnedActors()` returns
a snapshot from the maintained player-state ownership cache; retain weak actor references across
heartbeats and revalidate them before use. The subsystem owns its world lifetime and stops decisions
after the match result commits.

## Blueprint win conditions

Create your own GameMode Blueprint as a child of the generated starter GameMode
(`BP_GM_<ContentSetId>_Starter`) or of `RTSGameMode`, and save it outside the generated folders.
Then make a copy of the map outside the generated folders and select your GameMode in that copy's
**World Settings > GameMode Override**. The generated map selects the generated GameMode itself, so
changing only the project's **Maps & Modes** default does not override it. Do not add objective
logic directly to the generated GameMode or map; Generate replaces them:

- Call **Register Active Player** for every participant governed by last-player-standing resolution.
- Call **Eliminate Player** on authority when a custom objective, score, timer, surrender, or other
  player-scoped loss condition succeeds. It commits replicated defeat state exactly once, moves an
  eliminated human to observer, removes a registered participant, and resolves the final survivor.
- Call **Commit Match Result** on authority to name a winner directly. Pass no winner for a draw.
- Bind **On Player Defeated** and **On Match Ended** for presentation and game-specific aftermath;
  these events run after the replicated state is consistent. They report what happened; do not
  use them to make further match changes.

Both transaction nodes return `false` when called off authority, with an invalid/cross-world
controller, after the same state was already committed, or after the match ended. Treat `false` as
no state change.

## C++ win conditions

An authoritative objective system uses the same transactions:

```cpp
void AMyObjectiveGameMode::ResolveObjective(AController* CapturingPlayer)
{
	if (HasAuthority() && IsValid(CapturingPlayer))
	{
		CommitMatchResult(CapturingPlayer);
	}
}

void AMyObjectiveGameMode::HandleSurrender(AController* SurrenderingPlayer)
{
	if (HasAuthority())
	{
		EliminatePlayer(SurrenderingPlayer);
	}
}
```

`ARTSGameState` retains and replicates the defeated-player ledger and final result for current and
late-joining clients. Do not call its lower-level commit functions and separately notify GameMode;
the GameMode transactions keep observer transition, active-player removal, result resolution, and
Blueprint events ordered as one contract.

## Networking and lifetime

- Strategic AI and match-policy transactions execute only on the server.
- A client surrender or objective request must route through a validated server RPC owned by that
  client; never accept an arbitrary controller or winner reference from UI.
- AI policy callbacks stop when the authoritative GameMode reports the match over.
- Controllers, pawns, and objectives can disappear between heartbeats. Use `IsValid`, ownership,
  team, range, resource, cooldown, and current-state checks immediately before mutation.
- These APIs run on the game thread. Do not retain raw UObject pointers in asynchronous work; return
  to the game thread and resolve weak references before touching gameplay state.

`CPP_EXTENSION_CONTRACT.md` defines the shared async handoff, UObject lifetime, teardown, and
authoritative transaction rules for native policy implementations.
