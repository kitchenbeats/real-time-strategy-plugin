# Dedicated Server Lobbies

A generated map can run on a dedicated server, with a lobby that the server manages and no local
host player. In the Content Set's **Match Setup**, turn on **Require Player Setup**, add at least
one faction, keep **Human Controls First Base** on, and set **Minimum Human Players to Start**. Set
the number of bases and their factions in the Content Set, then choose **Generate Game**. Bases that
no human claims are played by the AI. This is not matchmaking, and players cannot take over a base
in the middle of a match.

## Player flow

Players press **Esc** on the setup screen, enter the server's address and port, and choose
**Join LAN game**. Each player chooses their own faction or **Spectate this match**, then clicks
**Ready**. The server starts only when the configured minimum number of human
participants is present and every assigned human has accepted the current lineup. A dedicated setup
always requires at least one human participant, even if the authored minimum is zero. Spectators do
not satisfy the minimum or block readiness. Changing the roster, factions, or base configuration
invalidates prior readiness; delayed requests for an older lineup are rejected.

Listen-host games keep their existing **Start Match** button. For administrator-controlled dedicated
games, disable the game mode's **Auto Start Skirmish** property. Players can still ready, but the
administrator must invoke the normal authoritative start operation. The interface explains this
policy instead of promising an automatic start.

After a match, an original participant can choose **Return to lobby**. This includes a defeated
human now viewing as an observer; it excludes people who joined as late spectators. The first
accepted request retires the old battlefield, clears the result, and reopens setup. Every human
must ready again. A return request contains the lineup revision, so a delayed request from a previous
match cannot reset a later result. Mid-match requests and duplicate resets are rejected.

If the last human participant disconnects, the dedicated server retires the abandoned battlefield
and reopens setup even when AI opponents remain. New arrivals can then claim bases. Spectator-only
departures do not reset a match. Engine shutdown and map travel do not publish competitive forfeits
or schedule abandonment resets. Existing spectator assignments stay explicit through rematches.

## Custom interfaces

Read `ARTSGameState::GetSkirmishLobbyState()`. `bServerManaged` identifies dedicated orchestration;
`bOpen`, `StartError`, `LineupRevision`, and the player/base entries describe the current state.
The local player's `ARTSPlayerController::RequestSkirmishLobbySelection` submits faction/readiness
changes. `RequestSkirmishLobbyReturn()` submits a postmatch return request. Both use owning-controller
RPCs; clients never choose the identity being modified. Rejections use the existing controller error
event. A successful submission is not proof of completion: consume the replicated lobby and match
state to update the interface.

The native result control refreshes when the lobby snapshot arrives, including when it replicates
after the final result. It does not infer participant eligibility from the current observer flag.

## Running a development server

A development server started from the editor executable can load the installed generated map
directly:

```sh
"<UE>/Engine/Binaries/Mac/UnrealEditor-Cmd" "<Project>/<Project>.uproject" \
  '/Game/RTSStarterGame/Generated/Maps/L_RTSComplete_Starter?MinHumanPlayers=2' \
  -server -Port=7777 -NullRHI -NoSound -unattended -stdout -FullStdOutLogOutput
```

Use the executable appropriate for the engine host platform. The map must be installed and its
plugin modules built for that engine. `MinHumanPlayers` is an existing authority launch override;
configure base/faction layouts in the authored scenario rather than relying on a URL base-count
shortcut. Clients run the normal game executable and join `server-address:7777` through the menu.
This is for development only; ship your game with a packaged dedicated-server build and test that.

Native tests under `RTS.Skirmish.Dedicated` exercise actual game instances, an ephemeral-port Unreal
listener, normal match startup, surrender, departure, and reset. The editor's real world-context
`RunAsDedicated` setting selects dedicated behavior. The existing listen-lobby tests remain separate.
For unattended development runs, `rts.lobby.ready 1`, `rts.lobby.surrender`, and
`rts.lobby.return` call the same local owner APIs as the interface; `rts.lobby.status` prints the
replicated result and participant state. These commands do not grant authority or bypass admission.

Before you ship, test with a real dedicated server process and at least two separately connected
clients: joining, readiness, rematches and players leaving. Also check what hidden information each
client receives, as described in [Network Security](NETWORK_SECURITY.md#release-verification).
