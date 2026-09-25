# Starter Audio

The starter sound palette lives in `/RealTimeStrategy/Audio/Starter`. Replace its
assignments in your project-owned HUD style to give the game your own sound identity.
The runtime resolves the owning local player's HUD style override first, then the
project's **RTS HUD Style** setting, then native defaults. A null sound means silence;
it does not inherit a sound from the bundled style.

| HUD style field | Native event |
| --- | --- |
| `ClickSound` | Accepted local order without a per-order entry |
| `OrderConfirmSounds` | Accepted order matching that order class, including Stop |
| `ErrorSound` | Local controller error, including rejected orders and supply failures |
| `AlertSound` | New under-attack event for an actor owned by this local player |
| `ToastSound` | Local production or research completion, cost refund, or visible node depletion |
| `VictorySound` | This local player wins |
| `DefeatSound` | This local player loses, including after becoming an observer |

An explicit null entry in `OrderConfirmSounds` mutes that order. Removing an entry
restores the generic acknowledgement. Disabling world markers through
`bEnableWorldCues` does not disable acknowledgement sounds. Authored sound concurrency
can limit overlapping cues; playback supplies the local controller as the concurrency
owner. Global volume remains controlled by the native game menu's audio setting.

Selection acknowledgement is a per-actor `URTSSelectableComponent.SelectedSound`
(`USoundCue`), independent of HUD style. For generated content, author it through
`ContentSet.WidgetDefaults.SelectedSound` and regenerate. The starter cue is
`/RealTimeStrategy/Audio/Starter/SC_RTS_Selected`, wrapping `SW_RTS_Selected`. Selection
plays only for the selecting local player's own actors and observes the existing
selection cooldown.

Each local-player cue subsystem subscribes to new events once. Refreshing a merged
toast or reconstructing a widget does not play the sound again. Supply failure does
not play both an error and a toast cue. Under-attack events retain the feed's configured
throttle. The final personal outcome sounds once per match; an in-place rematch resets
that state. Neutral spectators and draws show their existing result banner without a
personal victory or defeat sound. World travel and cleanup remove actor, controller,
and game-state subscriptions. Hidden resource depletion is not announced.

For remote active participants, resource-depletion feedback uses an owner-only reliable
notification selected by the server's per-player vision. It carries resource type and
location, so it can arrive after the source actor has been destroyed. The listen host's
local hidden flag does not determine what remote players can see. Local players retain
the existing source delegate path; `OnDepleted` itself is not multicast. A notification
is rejected after travel, a match-clock reset, a changed lobby lineup, match end, or
conversion to observer. No hidden-source event is sent for the UI to filter afterward.

The native `ARTSPlayerController` owns a replicated `URTSUpgradeComponent` named
`Player Upgrades` from construction, alongside its wallet. This lets the first live
research completion reach an already-bound local feed. AI and custom controller
classes retain the lazy ledger fallback. The feed follows replaced wallet/upgrade
components through its existing binding poll and detaches the previous objects.
It does not replay historical upgrade levels when presentation attaches late.

`RTS.UI.Audio.LocalRoutingAndLifetime` checks authored sound assets, real local-player
style ownership, gameplay event filtering, finite-resource depletion, match outcome
state, and world cleanup. It does not claim audible playback when run with `-nosound`.
To test on a real device, run the game with audio enabled and exercise selection,
an accepted order, a rejected order, damage, completion and a result. `Log LogRTS
Verbose` includes `[LocalAudio] submitted` records identifying the local player and
resolved asset. These records prove submission; an audible or recorded output review
is still required to verify device playback and mix quality.
