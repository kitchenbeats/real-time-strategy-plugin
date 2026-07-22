# AI and Match Policy Extension Contract

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

Derive from `ARTSPlayerAIController` and override the same two seams:

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

Create a customer-owned GameMode outside the generated output root by deriving the generated starter
GameMode or `RTSGameMode`, then select that class on a customer-owned map/copy outside the generated
output root. The generated starter map explicitly selects its generated GameMode, so changing only
the project's Maps & Modes default does not override it. Do not add objective logic directly to the
manifest-owned generated starter GameMode or map:

- Call **Register Active Player** for every participant governed by last-player-standing resolution.
- Call **Eliminate Player** on authority when a custom objective, score, timer, surrender, or other
  player-scoped loss condition succeeds. It commits replicated defeat state exactly once, moves an
  eliminated human to observer, removes a registered participant, and resolves the final survivor.
- Call **Commit Match Result** on authority to name a winner directly. Pass no winner for a draw.
- Bind **On Player Defeated** and **On Match Ended** for presentation and game-specific aftermath;
  these events run after replicated state is coherent and are notifications, not mutation seams.

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
